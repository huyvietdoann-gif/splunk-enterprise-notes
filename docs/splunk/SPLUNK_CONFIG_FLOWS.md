# Splunk configuration flows

Tài liệu này hệ thống các mối quan hệ chính giữa file cấu hình Splunk theo từng luồng hoạt động.

## props.conf
Thường dùng để:
- Chia luồng dữ liệu thành từng event.
- Nhận diện timestamp — thời gian xảy ra event.
- Quy định định dạng thời gian.
- Ghép nhiều dòng thành một event.
- Chọn cách xử lý dựa trên source, host hoặc sourcetype.
- Gọi cấu hình trong transforms.conf.

## transforms.conf
Thường thực hiện:
- Lọc bỏ event.
- Chuyển event sang index khác.
- Thay đổi metadata như host, source, sourcetype.
- Che dữ liệu nhạy cảm.
- Route dữ liệu — điều hướng dữ liệu đến đích khác.
- Trích xuất field khi search.

## 1. Thứ tự nạp cấu hình

```text
SPLUNK KHỞI ĐỘNG / RELOAD
          │
          ▼
Đọc và gộp cấu hình
          │
          ├── system/default/       thấp nhất
          ├── apps/<app>/default/
          ├── apps/<app>/local/
          └── system/local/         cao nhất
                      │
                      ▼
          EFFECTIVE CONFIGURATION
          cấu hình cuối cùng Splunk sử dụng
                      │
                      ▼
          Kiểm tra bằng btool --debug
```

## 2. Deployment Server quản lý Universal Forwarder

```text
WINDOWS UF
deploymentclient.conf
        │
        │ Tôi thuộc Deployment Server nào?
        │ targetUri=192.168.177.142:8089
        ▼
DEPLOYMENT SERVER
serverclass.conf
        │
        │ UF này thuộc nhóm nào?
        │ Phải nhận app nào?
        ▼
/opt/splunk/etc/deployment-apps/
        │
        ├── lab_windows_inputs/
        │       └── inputs.conf
        │
        └── lab_windows_outputs/
                └── outputs.conf
                    │
                    │ UF phone home và tải app
                    ▼
WINDOWS UF
C:\Program Files\SplunkUniversalForwarder\etc\apps\
        │
        ├── lab_windows_inputs/
        └── lab_windows_outputs/
                    │
                    ▼
          UF reload hoặc restart
```

```text
deploymentclient.conf → tìm Deployment Server
serverclass.conf       → ghép UF với app
deployment-apps        → chứa bản nguồn cấu hình
UF/etc/apps            → chứa bản đã được triển khai
```

## 3. Luồng thu thập và lưu dữ liệu

```text
WINDOWS EVENT LOG
System + Application
        │
        ▼
UF: inputs.conf
        │
        │ Đọc log nào?
        │ index=windows_lab
        ▼
UF: outputs.conf
        │
        │ Gửi tới đâu?
        │ 192.168.177.142:9997
        ▼
INDEXER: inputs.conf
        │
        │ Mở cổng nhận dữ liệu
        │ [splunktcp://9997]
        ▼
INDEXER: props.conf
        │
        │ Chia event, lấy timestamp,
        │ xác định sourcetype
        ▼
INDEXER: transforms.conf
        │
        │ Lọc, route, đổi metadata,
        │ che dữ liệu nếu cần
        ▼
INDEXER: indexes.conf
        │
        │ Index có tồn tại không?
        │ Lưu ở đâu và giữ bao lâu?
        ▼
$SPLUNK_DB/windows_lab/
        │
        ├── db/          Hot + Warm bucket
        ├── colddb/      Cold bucket
        └── thaweddb/    Dữ liệu phục hồi
```

```text
Nguồn log
→ inputs.conf
→ outputs.conf
→ receiver inputs.conf
→ props.conf
→ transforms.conf
→ indexes.conf
→ bucket
```

## 4. Luồng có Heavy Forwarder

```text
NGUỒN LOG
    │
    ▼
UF
├── inputs.conf
└── outputs.conf
    │
    ▼
HEAVY FORWARDER
├── inputs.conf
├── props.conf
├── transforms.conf
└── outputs.conf
    │
    │ dữ liệu đã parse
    ▼
INDEXER
├── inputs.conf
└── indexes.conf
```

```text
UF → Indexer
→ parsing config đặt trên Indexer

UF → HF → Indexer
→ parsing config đặt trên HF
```

## 5. Quan hệ giữa props.conf và transforms.conf

### Index-time

```text
EVENT ĐI VÀO
     │
     ▼
props.conf
     │
     │ Event thuộc sourcetype nào?
     │ Gọi transform nào?
     ▼
transforms.conf
     │
     ├── Giữ hoặc loại event
     ├── Đổi index
     ├── Đổi host/sourcetype
     ├── Route sang output khác
     └── Mask dữ liệu
              │
              ▼
         indexes.conf
              │
              ▼
          LƯU EVENT
```

### Search-time

```text
USER CHẠY SEARCH
        │
        ▼
SH: props.conf
        │
        ├── EXTRACT
        ├── REPORT
        ├── LOOKUP
        ├── FIELDALIAS
        └── EVAL
               │
               ▼
SH: transforms.conf
        │
        ├── Regex tách field
        └── Định nghĩa lookup
               │
               ▼
      FIELD HIỂN THỊ TRONG SEARCH
```

## 6. Luồng tìm kiếm

```text
USER
 │
 │ mở Splunk Web:8000
 ▼
SH: web.conf
 │
 │ Splunk Web có chạy không?
 │ HTTP hay HTTPS?
 ▼
SH: authentication.conf
 │
 │ User xác thực bằng Splunk, LDAP hay SAML?
 ▼
SH: authorize.conf
 │
 │ User có role nào?
 │ Được search index nào?
 ▼
SH: savedsearches.conf / macros.conf
 │  props.conf / transforms.conf
 │
 │ Chuẩn bị SPL và search-time field
 ▼
SH: distsearch.conf
 │
 │ Gửi search tới Indexer nào?
 ▼
INDEXER:8089
 │
 │ Đọc bucket và chạy phần search
 ▼
SEARCH HEAD
 │
 │ Gom kết quả
 ▼
USER
```

## 7. Indexer Cluster

### Quản lý cấu hình peer

```text
CLUSTER MANAGER
/opt/splunk/etc/manager-apps/<app>/local/
        │
        ├── indexes.conf
        ├── props.conf
        ├── transforms.conf
        └── app cho peer
        │
        │ apply cluster-bundle
        ▼
INDEXER PEERS
├── Peer 1
├── Peer 2
└── Peer 3
```

### Luồng dữ liệu và replication

```text
UF: outputs.conf
        │
        │ load balancing qua 9997
        ├────────→ Peer 1
        ├────────→ Peer 2
        └────────→ Peer 3
                       │
                       │ một peer nhận dữ liệu
                       ▼
                CLUSTER MANAGER
                kiểm tra số bản sao
                       │
                       ▼
             PEER ↔ PEER qua 9887
                 replication dữ liệu
```

Event không đi qua Cluster Manager.

## 8. Indexer Discovery

```text
UF: outputs.conf
        │
        │ manager_uri=Cluster Manager:8089
        ▼
CLUSTER MANAGER
        │
        │ Trả danh sách peer đang hoạt động
        ▼
UF
        │
        ├──→ Peer 1:9997
        ├──→ Peer 2:9997
        └──→ Peer 3:9997
```

```text
Deployment Server → gửi outputs.conf cho UF
Cluster Manager   → cung cấp danh sách peer
UF                → tự gửi dữ liệu trực tiếp tới peer
```

## 9. Search Head Cluster

### Cấu hình thành viên

```text
SHC MEMBERS
server.conf
        │
        │ [shclustering]
        ├── SH1
        ├── SH2
        └── SH3
             │
             ▼
Một thành viên được bầu làm Captain
```

### Triển khai app quản trị

```text
SHC DEPLOYER
/opt/splunk/etc/shcluster/apps/
        │
        │ apply shcluster-bundle
        ▼
SEARCH HEAD CLUSTER
├── SH1
├── SH2
└── SH3
```

### Nội dung user tạo

```text
USER TẠO DASHBOARD TRÊN SH1
        │
        ▼
SHC REPLICATION
        │
        ├──→ SH2
        └──→ SH3
```

Deployer phân phối app quản trị; SHC replication đồng bộ nội dung do user tạo.

## 10. Port và file điều khiển

```text
User ──8000──→ Splunk Web
               web.conf

UF ──9997──→ Indexer receiver
outputs.conf  inputs.conf

UF ──8089──→ Deployment Server
deploymentclient.conf + serverclass.conf

Search Head ──8089──→ Indexer
distsearch.conf       server.conf

Ứng dụng ──8088──→ HEC
                    inputs.conf

Peer ──9887──→ Peer
server.conf [clustering]
```

## 11. Flow xử lý lỗi không có dữ liệu

```text
SEARCH KHÔNG CÓ EVENT
        │
        ▼
Nguồn có tạo log không?
        │
        ├── Không → sửa nguồn
        └── Có
             ▼
UF inputs.conf có enable không?
             │
             ├── Không → sửa inputs.conf
             └── Có
                  ▼
UF btool inputs có thấy stanza không?
                  │
                  ├── Không → kiểm tra app/precedence
                  └── Có
                       ▼
outputs.conf có đúng Indexer:9997 không?
                       │
                       ├── Không → sửa outputs.conf
                       └── Có
                            ▼
Indexer có listen 9997 không?
                            │
                            ├── Không → sửa Indexer inputs.conf
                            └── Có
                                 ▼
Index có tồn tại không?
                                 │
                                 ├── Không → sửa indexes.conf
                                 └── Có
                                      ▼
Search trực tiếp trên Indexer có thấy không?
                                      │
                         ┌────────────┴────────────┐
                        Không                     Có
                         │                         │
                         ▼                         ▼
              props/transforms/log         Lỗi phía Search Head
              quyền ghi/index              distsearch.conf
                                           authorize.conf
                                           time range
```

## 12. Ghi nhớ nhanh: chức năng và đường dẫn

`<app>` là tên app chứa cấu hình. Với app do Deployment Server quản lý, sửa bản nguồn trong `deployment-apps`, không sửa bản đã nhận trên UF.

| File | Chức năng | Máy sử dụng | Đường dẫn thường dùng |
|---|---|---|---|
| `inputs.conf` | Lấy dữ liệu vào | Windows UF | `C:\Program Files\SplunkUniversalForwarder\etc\apps\<app>\default\inputs.conf` |
| `inputs.conf` | Mở cổng nhận dữ liệu `9997` | Indexer | `/opt/splunk/etc/apps/<app>/local/inputs.conf` — lab: `/opt/splunk/etc/apps/launcher/local/inputs.conf` |
| `outputs.conf` | Gửi dữ liệu tới Indexer hoặc HF | Windows UF | `C:\Program Files\SplunkUniversalForwarder\etc\apps\<app>\default\outputs.conf` |
| `deploymentclient.conf` | UF tìm Deployment Server | Windows UF | `C:\Program Files\SplunkUniversalForwarder\etc\system\local\deploymentclient.conf` |
| `serverclass.conf` | Chọn UF thuộc nhóm nào và nhận app nào | Deployment Server | `/opt/splunk/etc/system/local/serverclass.conf` |
| `props.conf` | Chia event, nhận timestamp và gọi transform | Indexer hoặc HF | `/opt/splunk/etc/apps/<app>/local/props.conf` |
| `props.conf` | Trích xuất field lúc search | Search Head | `/opt/splunk/etc/apps/<app>/local/props.conf` |
| `transforms.conf` | Lọc, che hoặc route dữ liệu | Indexer hoặc HF | `/opt/splunk/etc/apps/<app>/local/transforms.conf` |
| `transforms.conf` | Trích xuất field lúc search | Search Head | `/opt/splunk/etc/apps/<app>/local/transforms.conf` |
| `indexes.conf` | Quy định nơi lưu, dung lượng và thời gian giữ dữ liệu | Indexer | `/opt/splunk/etc/apps/<app>/local/indexes.conf` — lab: `/opt/splunk/etc/apps/search/local/indexes.conf` |
| `distsearch.conf` | Search Head khai báo các Indexer để search | Search Head | `/opt/splunk/etc/system/local/distsearch.conf` |
| `web.conf` | Cấu hình Splunk Web | Splunk Enterprise | `/opt/splunk/etc/system/local/web.conf` |
| `server.conf` | Cấu hình instance, TLS và cluster | Splunk Enterprise | `/opt/splunk/etc/system/local/server.conf` |
| `limits.conf` | Giới hạn tài nguyên và hoạt động search | Search Head hoặc Indexer | `/opt/splunk/etc/system/local/limits.conf` |

### Đường dẫn khi triển khai tập trung

```text
Deployment Server quản lý UF
/opt/splunk/etc/deployment-apps/<app>/default/<file>.conf
        │
        └── triển khai tới UF:
            C:\Program Files\SplunkUniversalForwarder\etc\apps\<app>\default\<file>.conf

Cluster Manager quản lý Indexer peers
/opt/splunk/etc/manager-apps/<app>/local/<file>.conf

SHC Deployer quản lý Search Head Cluster
/opt/splunk/etc/shcluster/apps/<app>/local/<file>.conf
```

Không chỉnh file trong `etc/system/default/` hoặc `etc/apps/<app>/default/` của app cài sẵn. Đặt cấu hình tùy chỉnh trong `local/` hoặc trong app riêng.

## 13. Lab hiện tại

```text
WINDOWS UF: DESKTOP-TSA3GHF
deploymentclient.conf
        │ 8089
        ▼
hvd-lab: Deployment Server
serverclass.conf
        │
        ├── lab_windows_inputs
        └── lab_windows_outputs
                    │
                    ▼
WINDOWS UF
inputs.conf: System + Application
outputs.conf: 192.168.177.142:9997
                    │
                    ▼
hvd-lab: Indexer
inputs.conf: [splunktcp://9997]
indexes.conf: [windows_lab]
                    │
                    ▼
Search Head riêng
distsearch.conf: 192.168.177.142:8089
                    │
                    ▼
User search index=windows_lab
```

