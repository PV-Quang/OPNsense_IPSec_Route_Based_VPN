# Cấu hình IPsec Site-to-Site Route-Based VPN trên OPNsense

Tài liệu này hướng dẫn cấu hình **IPsec Site-to-Site VPN dạng Route-Based sử dụng VTI (Virtual Tunnel Interface)** trên OPNsense, kết nối tới một peer VPN chạy trên **VMware Cloud Director / NSX Edge Gateway**.

> [!IMPORTANT]
> Các địa chỉ IP và tên object trong tài liệu là thông số của môi trường lab. Khi áp dụng production, hãy thay bằng thông số thực tế của hệ thống.
>
> Không đưa **Pre-Shared Key (PSK)** thật lên GitHub. Ảnh PSK trong repository này đã được che giá trị.

---

## 1. Mô hình kết nối

```mermaid
flowchart LR
    A[POC LAN\n30.0.0.0/24] --> B[OPNsense\nLAN: 30.0.0.1\nWAN: 61.14.236.216]
    B --> C[VTI\n169.254.123.1/30]
    C <-->|IKEv2 / IPsec| D[VTI\n169.254.123.2/30]
    D --> E[VCD / NSX Edge\nWAN: 103.141.177.86]
    E --> F[PROD LAN\n20.0.0.0/24]
```

### IP plan sử dụng trong lab

| Thành phần | Site POC - OPNsense | Site PROD - VCD Edge |
|---|---|---|
| WAN/Public IP | `61.14.236.216` | `103.141.177.86` |
| LAN | `30.0.0.0/24` | `20.0.0.0/24` |
| VTI IP | `169.254.123.1` | `169.254.123.2` |
| VTI subnet | `169.254.123.0/30` | `169.254.123.0/30` |
| Remote LAN cần route | `20.0.0.0/24` | `30.0.0.0/24` |

### Crypto profile phía peer

| Tham số | Giá trị |
|---|---|
| IKE Version | IKEv2 |
| IKE Encryption | AES-256 |
| IKE Integrity/Digest | SHA-256 |
| Diffie-Hellman | Group 14 / MODP 2048 |
| IKE SA Lifetime | 86400 giây |
| ESP Encryption | AES-256 |
| ESP Integrity/Digest | SHA-256 |
| PFS | Group 14 |
| Child SA Lifetime | 3600 giây |
| Authentication | Pre-Shared Key |

> [!NOTE]
> `DPD delay` và `SA lifetime/rekey time` là hai tham số khác nhau. Không dùng trường DPD để thay thế cho IKE/Child SA lifetime. Nếu giao diện OPNsense của bạn expose các trường lifetime/rekey riêng, hãy cấu hình khớp với peer hoặc giữ default nếu hai bên đã negotiate ổn định.

---

## 2. Prerequisites

Trước khi cấu hình, cần đảm bảo:

- OPNsense WAN có Internet/reachability tới peer public IP `103.141.177.86`.
- Peer VCD Edge có reachability tới OPNsense WAN `61.14.236.216`.
- UDP `500` và `4500` được phép giữa hai public IP.
- Nếu không dùng NAT-T, ESP (IP protocol 50) cũng phải được phép trên đường truyền.
- Hai LAN không overlap với nhau hoặc với subnet/VIP/route khác trong hệ thống.
- Pre-Shared Key giống nhau ở hai đầu.
- Route-Based VPN phía peer đã có VTI `169.254.123.2/30`.

---

# 3. Cấu hình trên OPNsense

## Bước 1 - Tạo Virtual Tunnel Interface (VTI)

Vào:

```text
VPN → IPsec → Virtual Tunnel Interfaces
```

Tạo VTI với thông số:

| Field | Value |
|---|---|
| Enabled | Enable |
| Reqid | `10` |
| Local address | `61.14.236.216` |
| Remote address | `103.141.177.86` |
| Tunnel local address | `169.254.123.1` |
| Tunnel remote address | `169.254.123.2` |
| Skip firewall rules | Disable |
| Name | `VTI-PROD` |

![VTI configuration](assets/01-vti-config.png)

**Điểm quan trọng:** `Reqid` phải trùng với `Reqid` của Child SA ở bước Phase 2.

---

## Bước 2 - Tạo Pre-Shared Key

Vào:

```text
VPN → IPsec → Pre-Shared Keys
```

Cấu hình:

| Field | Value |
|---|---|
| Local Identifier | `61.14.236.216` |
| Remote Identifier | `103.141.177.86` |
| Pre-Shared Key | `<YOUR-PSK>` |
| Type | `PSK` |

![Pre-Shared Key](assets/02-pre-shared-key-redacted.png)

> [!WARNING]
> Không commit PSK thật vào Git repository. Nếu repository đã từng chứa PSK, hãy **rotate PSK**, không chỉ xóa khỏi commit mới.

---

## Bước 3 - Tạo IKE Connection / Phase 1

Vào:

```text
VPN → IPsec → Connections
```

Tạo connection, ví dụ `PROD-EDGE-P1`:

| Field | Value |
|---|---|
| Enabled | Enable |
| Proposal | `aes256-sha256-modp2048 [DH14]` |
| Version | `IKEv2` |
| MOBIKE | Disable |
| Local addresses | `61.14.236.216` |
| Remote addresses | `103.141.177.86` |
| Description | `PROD-EDGE-P1` |

![IKE Connection](assets/03-ike-connection.png)

### Giải thích proposal

```text
aes256-sha256-modp2048
```

Tương ứng:

```text
AES-256
SHA-256
DH Group 14 / MODP 2048
```

Đây phải khớp với Security Profile của peer.

---

## Bước 4 - Cấu hình Local Authentication

Trong Connection vừa tạo, thêm **Local Authentication**:

| Field | Value |
|---|---|
| Enabled | Enable |
| Connection | `PROD-EDGE-P1` |
| Round | `0` |
| Authentication | `Pre-Shared Key` |
| ID | `61.14.236.216` |
| Description | `OPNSense-POC` |

![Local Authentication](assets/04-local-authentication.png)

---

## Bước 5 - Cấu hình Remote Authentication

Thêm **Remote Authentication**:

| Field | Value |
|---|---|
| Enabled | Enable |
| Connection | `PROD-EDGE-P1` |
| Round | `0` |
| Authentication | `Pre-Shared Key` |
| ID | `103.141.177.86` |
| Description | `VCD-EDGE-PROD` |

![Remote Authentication](assets/05-remote-authentication.png)

---

## Bước 6 - Tạo Child SA / Phase 2

Trong Connection `PROD-EDGE-P1`, thêm Child SA:

| Field | Value |
|---|---|
| Enabled | Enable |
| Mode | `Tunnel` |
| Policies | **Disable / unchecked** |
| Start Action | `Trap` |
| DPD Action | `Clear` |
| Reqid | `10` |
| ESP proposals | `aes256-sha256-modp2048 [DH14]` |
| Local | `0.0.0.0/0` |
| Remote | `0.0.0.0/0` |
| Description | `PROD-EDGE-P2` |

![Child SA](assets/06-child-sa.png)

### Vì sao Local/Remote là `0.0.0.0/0`?

Với **Route-Based IPsec**, việc quyết định subnet nào đi qua tunnel được xử lý bởi **routing table**, không phải policy selector kiểu Policy-Based VPN.

Do đó:

```text
Child SA: 0.0.0.0/0 ↔ 0.0.0.0/0
Routing : 20.0.0.0/24 → VTI peer 169.254.123.2
```

**Policies phải bỏ tick** để traffic được forward theo VTI/routing thay vì cài policy selector vào kernel.

---

## Bước 7 - Assign VTI thành interface

Vào:

```text
Interfaces → Assignments
```

Sau khi VTI được tạo đúng, OPNsense sẽ sinh interface dạng `ipsec<Reqid>`.

Với `Reqid = 10`:

```text
ipsec10
```

Assign interface này và đặt Description là `VTIPROD`.

![Interface assignment](assets/07-interface-assignment.png)

Mapping trong lab:

```text
ipsec10 → VTIPROD → VTI-PROD
```

---

## Bước 8 - Tạo Gateway cho VTI

Vào:

```text
System → Gateways → Configuration
```

Tạo gateway:

| Field | Value |
|---|---|
| Name | `PROD_VTI_G` |
| Interface | `VTIPROD` |
| Address Family | `IPv4` |
| IP Address | `169.254.123.2` |
| Upstream Gateway | Disable |

![VTI Gateway](assets/08-vti-gateway.png)

Gateway này chính là **VTI IP phía peer**.

---

## Bước 9 - Tạo Static Route tới remote LAN

Vào:

```text
System → Routes → Configuration
```

Tạo route:

```text
Network : 20.0.0.0/24
Gateway : PROD_VTI_G - 169.254.123.2
```

![Static Route](assets/09-static-route.png)

Sau khi Apply, kiểm tra routing table từ OPNsense shell:

```sh
netstat -rn4
```

Kết quả cần có route tương tự:

```text
20.0.0.0/24    169.254.123.2    UGS    ipsec10
169.254.123.2   link#...         UH     ipsec10
```

![Routing table](assets/12-routing-table.png)

---

## Bước 10 - Tạo Firewall Rules

### 10.1 LAN → PROD

Nếu LAN đang có default rule:

```text
LAN net → any : Pass
```

thì traffic `30.0.0.0/24 → 20.0.0.0/24` đã được phép.

Trong production nên thu hẹp rule theo đúng source/destination/service cần thiết.

### 10.2 PROD → POC qua IPsec

Trên OPNsense phiên bản mới, traffic VTI mặc định được filter tại interface group:

```text
IPsec encapsulation
```

Tạo rule:

```text
Action      : Pass
Direction   : In
IP Version  : IPv4
Protocol    : any
Source      : 20.0.0.0/24
Destination : 30.0.0.0/24
Gateway     : None
```

Nếu cần traffic hai chiều rõ ràng, có thể tạo thêm rule ngược lại theo policy vận hành.

![Firewall Rules](assets/10-firewall-rules.png)

> [!IMPORTANT]
> Không chỉ tạo rule trên interface `VTIPROD`. Với default IPsec filtering behavior của OPNsense, hãy kiểm tra rule tại **IPsec encapsulation**.

---

# 4. Cấu hình cần có ở peer VCD / NSX Edge

Tài liệu này tập trung vào OPNsense, nhưng phía peer cần tối thiểu:

```text
Type           : Route Based
Local WAN      : 103.141.177.86
Remote WAN     : 61.14.236.216
Local VTI      : 169.254.123.2/30
Remote VTI     : 169.254.123.1/30
Static Route   : 30.0.0.0/24 via 169.254.123.1
IKE            : IKEv2 / AES256 / SHA256 / DH14
ESP            : AES256 / SHA256 / DH14
Authentication : PSK
```

### Peer endpoint reference

![VCD Edge endpoint reference](assets/16-vcd-edge-endpoint-reference.png)

### Peer security profile reference

![VCD Edge security profile](assets/17-vcd-edge-security-profile-reference.png)

---

## 4.1 NAT Exemption / NO_SNAT phía VCD Edge

Nếu PROD subnet có Internet SNAT dạng:

```text
20.0.0.0/24 → 103.141.177.86
```

hãy tạo một `NO_SNAT` có priority cao hơn cho VPN traffic:

```text
Action      : NO_SNAT
Source      : 20.0.0.0/24
Destination : 30.0.0.0/24
```

![VCD NO_SNAT](assets/13-vcd-no-snat.png)

Mục tiêu là giữ nguyên packet:

```text
20.0.0.251 → 30.0.0.200
```

thay vì bị translate thành public IP.

---

# 5. Kiểm tra trạng thái VPN

Vào:

```text
VPN → IPsec → Status Overview
```

Cần thấy:

```text
Phase 1 / IKE SA   : Established / Up
Phase 2 / Child SA : INSTALLED
```

![IPsec Status](assets/11-ipsec-status.png)

Tunnel **UP** chỉ xác nhận control-plane IPsec đã thiết lập. LAN-to-LAN còn phụ thuộc:

- Static route
- Firewall rule
- NAT exemption
- Return route
- Endpoint firewall
- Không có route/subnet overlap

---

# 6. Connectivity Test

Trong lab này, dùng test VM POC `30.0.0.200` và VM PROD `20.0.0.251`.

Từ POC:

```cmd
ping 20.0.0.251
```

Từ PROD:

```cmd
ping 30.0.0.200
```

Có thể kiểm tra VTI trước:

```text
169.254.123.1 → 169.254.123.2
```

Nếu ping host Windows, kiểm tra Windows Firewall/ICMP rule trước khi kết luận VPN lỗi.

---

# 7. Troubleshooting Checklist

## 7.1 Tunnel UP nhưng LAN không ping được

Kiểm tra theo thứ tự:

1. `VPN → IPsec → Status Overview`
2. Routing table trên OPNsense
3. Firewall `IPsec encapsulation`
4. VCD Edge Gateway Firewall
5. NO_SNAT / SNAT priority
6. Windows/Linux host firewall
7. Packet capture tại `ipsec10`
8. NSX Traceflow nếu peer là NSX/VCD
9. Kiểm tra route overlap / more-specific route

---

## 7.2 Kiểm tra route trên OPNsense

```sh
netstat -rn4
```

Route đúng phải có:

```text
20.0.0.0/24 → 169.254.123.2 → ipsec10
```

Nếu route remote LAN không tồn tại, packet sẽ đi default route/WAN thay vì VTI.

---

## 7.3 Packet capture trên VTI

Capture ICMP giữa hai endpoint:

```sh
tcpdump -ni ipsec10 'icmp and host 30.0.0.200 and host 20.0.0.251'
```

Kết quả bình thường phải thấy cả hai chiều:

```text
30.0.0.200 > 20.0.0.251: ICMP echo request
20.0.0.251 > 30.0.0.200: ICMP echo reply
```

Nếu chỉ thấy request mà không có reply:

```text
POC → VPN → PROD = OK
Return path PROD → VPN → POC = có vấn đề
```

Lúc đó tập trung kiểm tra peer-side route, NAT, firewall và endpoint.

---

# 8. Case Study - Route conflict với NSX Advanced Load Balancer/Avi

Trong quá trình triển khai lab, ban đầu test VM POC dùng:

```text
30.0.0.100
```

Hiện tượng:

```text
30.0.0.100 → 20.0.0.251 : FAIL
20.0.0.251 nhận Echo Request  : YES
20.0.0.251 tạo Echo Reply     : YES
OPNsense không nhận Echo Reply: YES
```

NSX Traceflow từ:

```text
20.0.0.251 → 30.0.0.100
```

cho thấy packet được **Delivered tới Avi Service Engine** thay vì quay về IPsec VTI.

![NSX Traceflow route conflict](assets/14-nsx-traceflow-route-conflict.png)

Search `30.0.0.100` trên NSX cũng trả về một static route liên quan `avi-nsx-VCD-LoadBalancer`:

![NSX static route search](assets/15-nsx-search-static-route.png)

### Root cause

Trong môi trường PROD đã tồn tại một **more-specific route** liên quan tới `30.0.0.100` do NSX Advanced Load Balancer/Avi tạo. Vì route đó cụ thể hơn route VPN `30.0.0.0/24`, return traffic được đưa tới Avi thay vì VTI.

Routing luôn ưu tiên **Longest Prefix Match**:

```text
More-specific host/VIP route  >  30.0.0.0/24 VPN route
```

### Xác nhận RCA

Đổi IP test VM từ:

```text
30.0.0.100
```

sang:

```text
30.0.0.200
```

sau đó ping qua VPN thành công.

### Khuyến nghị

Trước khi chọn remote subnet cho Site-to-Site VPN, kiểm tra:

- Connected subnet trên T0/T1
- Static route
- NAT/VIP subnet
- Avi/NSX ALB VIP
- Kubernetes/LB service subnet
- Existing customer tenant subnet

Tránh dùng một subnet đang được sử dụng hoặc có more-specific routes ở site đối diện.

> [!TIP]
> `20.0.0.0/24` và `30.0.0.0/24` không thuộc RFC1918 private address space. Với mạng internal/lab mới, ưu tiên các dải `10.0.0.0/8`, `172.16.0.0/12` hoặc `192.168.0.0/16` để giảm nguy cơ conflict với public address space.

---

# 9. Verification Checklist

Sau khi hoàn tất, xác nhận toàn bộ checklist sau:

- [ ] WAN hai đầu reach được nhau
- [ ] PSK giống nhau
- [ ] IKEv2 proposal khớp
- [ ] ESP proposal khớp
- [ ] VTI local/remote đúng chiều
- [ ] VTI `Reqid` trùng Child SA `Reqid`
- [ ] Child SA `Policies` disabled
- [ ] `ipsec10` đã assign thành interface
- [ ] Gateway VTI trỏ tới peer `169.254.123.2`
- [ ] Static route `20.0.0.0/24 → 169.254.123.2`
- [ ] Firewall `IPsec encapsulation` allow đúng source/destination
- [ ] Peer có static route return về `30.0.0.0/24`
- [ ] Peer có NO_SNAT cho VPN traffic nếu tồn tại Internet SNAT
- [ ] Phase 1 UP
- [ ] Child SA INSTALLED
- [ ] `netstat -rn4` hiển thị route qua `ipsec10`
- [ ] Packet capture thấy request và reply
- [ ] Không có subnet/route conflict

---

# 10. Security / Production Notes

- Chỉ allow đúng subnet và service cần thiết, tránh `any-any` lâu dài.
- PSK nên đủ dài và random; rotate định kỳ theo policy.
- Không lưu PSK trong Git, ticket công khai hoặc screenshot không che.
- Bật logging có chọn lọc trong thời gian troubleshoot; tránh bật log quá rộng lâu dài.
- Trước khi thay đổi route/NAT production, chuẩn bị rollback plan.
- Khi tunnel UP nhưng traffic fail, **không nên liên tục thay đổi Phase 1/Phase 2** nếu SA đã established; ưu tiên kiểm tra data plane: route → firewall → NAT → return path → packet capture.

---

## Repository layout

```text
OPNsense_IPSec_Route_Based_VPN/
├── README.md
└── assets/
    ├── 01-vti-config.png
    ├── 02-pre-shared-key-redacted.png
    ├── 03-ike-connection.png
    ├── 04-local-authentication.png
    ├── 05-remote-authentication.png
    ├── 06-child-sa.png
    ├── 07-interface-assignment.png
    ├── 08-vti-gateway.png
    ├── 09-static-route.png
    ├── 10-firewall-rules.png
    ├── 11-ipsec-status.png
    ├── 12-routing-table.png
    ├── 13-vcd-no-snat.png
    ├── 14-nsx-traceflow-route-conflict.png
    ├── 15-nsx-search-static-route.png
    ├── 16-vcd-edge-endpoint-reference.png
    └── 17-vcd-edge-security-profile-reference.png
```

---

## Ghi chú phiên bản

Screenshot trong tài liệu được thực hiện trên OPNsense UI thế hệ mới, trong đó firewall interface cho IPsec có thể hiển thị dưới tên **`IPsec encapsulation`**. Tên menu/field có thể thay đổi nhẹ giữa các phiên bản OPNsense.
