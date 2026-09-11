1. https://tdx.transportdata.tw/api/basic/v1/EV/ConnectorLiveStatus/Tourism

可以取得 充電槍 即時狀態

{

      "StationID": "89132567-S0004",

      "ChargingPointID": "89132567-P0007",

      "ConnectorID": "89132567-C0007",

      "ConnectorType": 5,

      "ConnectorStatus": 3,

      "ChargedKWH": 0,

      "ChargedPercent": 0,

      "LastUpdateTime": "2026-09-11T20:50:00+08:00"

    },

2. https://tdx.transportdata.tw/api/basic/v1/EV/Station/Tourism

可以取得 充電站基本資料

{

      "StationID": "89132567-S0002",

      "StationName": {

        "Zh_tw": "Acon-eco台南井仔腳鹽田站",

        "En": "AconElectricEnergyStation"

      },

      "Description": "井仔腳停車場",

      "OperatorID": "89132567",

      "OperationType": 2,

      "PositionLat": 23.25938,

      "PositionLon": 120.10868,

      "Geometry": "POLYGON ((120.108 23.259, 120.109 23.259, 120.109 23.259, 120.108 23.259, 120.108 23.259))",

      "Spaces": 2,

      "ChargingPoints": 2,

      "Connectors": [

        {

          "Type": 5,

          "Power": 1,

          "Quantity": 2

        }

      ],

      "ServiceTime": "7*24小時",

      "ParkingRate": "免費",

      "ChargingRate": "計度/8元每度",

      "Floors": "1",

      "UsageRestriction": "依現場操作說明告示",

      "PhotoURLs": [

        "https://ev-acon.lifeplus.tw/image/tainanyenten.jpg"

      ],

      "Location": {

        "Place": {

          "POI": "井仔腳停車場"

        }

      },

      "Telephone": "0800-868885"

    },

我現在收到的有這個資料內容，幫我整每個欄位對應的意義，然後我可以做出

1. 知道哪個 station 的充電站的每天使用狀況

2. 知道那個 station的充電站的壞掉時數累積（每天，每週，每月）

這兩份資料剛好構成了電動車充電管理系統最經典的架構：「靜態站點維度資料」結合「動態即時狀態資料」。一、欄位對應與意義整理1. 即時狀態資料 (ConnectorLiveStatus)此 API 提供單一充電槍在當下時間點的快照（Snapshot）：欄位名稱範例值意義與業務用途StationID89132567-S0004所屬充電站編號，用來關聯基本資料（Foreign Key）。ChargingPointID89132567-P0007所屬充電樁（機柱）編號。一樁可能有多槍。ConnectorID89132567-C0007充電槍唯一代碼。計算使用與故障時數的最小追蹤單元。ConnectorType5槍頭規格碼（5 代表 CCS1 美規快充）。ConnectorStatus3槍頭當前狀態碼：• 1：Available（空閒可用）• 2：Occupied / Charging（使用中/充電中）• 3：Reserved（預約）或特定維護狀態• 4：Faulted / OutOfService（故障/停止服務）• 0 或 255：Unknown（離線/未知）ChargedKWH0當次充電累積度數（kWh）。ChargedPercent0當前車輛電池充電進度（%）。LastUpdateTime2026-09-11T20:50:00+08:00該槍設備端最後狀態回報時間。2. 充電站基本資料 (Station)此 API 提供充電站的地理位置與設備規模：欄位名稱範例值意義與業務用途StationID89132567-S0002站點唯一編號（Primary Key）。StationNameZh*tw: 井仔腳鹽田站站點中英文名稱，前端展示與報表標題用。PositionLat / PositionLon23.25938 / 120.10868站點經緯度，用於地圖標記（Map Marker）。Spaces2充電專用停車位總數。ChargingPoints2站內充電樁總座數。ConnectorsType: 5, Power: 1, Quantity: 2站內槍頭統計（此站共有 2 支 CCS1 槍）。ServiceTime7\*24小時營業時間，用來判斷故障時數是否需排除非營業時段。ParkingRate / ChargingRate停車免費 / 8元每度計費規則說明。二、如何實現「站點每日使用狀況」？要計算站點的每日使用時數與使用率，核心是追蹤槍頭狀態為 2（充電中）的時間區間。1. 資料收集與事件化（以 Connector 為單位）輪詢排程：每 1～2 分鐘呼叫一次 ConnectorLiveStatus。記錄狀態日誌：當 ConnectorID 的狀態從非 2 變成 2 時，記錄 StartChargingTime；當狀態從 2 變回其他狀態時，記錄 EndChargingTime。單次使用時長：$$\Delta t = \text{EndChargingTime} - \text{StartChargingTime}$$2. 彙總到站點維度（Station Aggregation）每天 23:59:59 進行日結統計：站點每日總充電時數：$$\text{StationDailyUsageHours} = \sum*{\text{站內所有槍}} \text{當天使用時數}$$每日設備使用率（Utilization Rate）：$$\text{使用率} = \frac{\text{StationDailyUsageHours}}{\text{站內總槍數} \times 24} \times 100\%$$每日翻轉次數（Turnover）：當天該站所有槍狀態轉為 2 的總次數（代表今天服務了多少台車）。三、如何實現「站點故障時數累積（天 / 週 / 月）」？故障分析追蹤的是異常狀態（如 4: Faulted 或設備無訊號超時）的持續時間。1. 定義「故障事件」（Fault Event）在資料庫中維護一張 connector_fault_logs 表，包含欄位：ConnectorID、StationID、FaultStartTime、FaultEndTime、DurationMinutes。觸發條件包括：明確故障：ConnectorStatus 變更為 4（故障）或營運商定義的維修碼。設備離線（心跳超時）：LastUpdateTime 超過閥值（例如超過 30 分鐘沒有更新，判定為設備斷網/離線）。2. 跨週期時數累積統計（Rollup）透過 SQL 或後端排程定期彙總（若故障跨天，需依當日 00:00～24:00 切分）：每日故障時數：$$\text{DailyFaultHours} = \sum \text{當天故障區間時數}$$每週 / 每月累計故障時數：$$\text{WeeklyFaultHours} = \sum*{\text{本週 7 天}} \text{DailyFaultHours}$$$$\text{MonthlyFaultHours} = \sum*{\text{本月各天}} \text{DailyFaultHours}$$可用度指標（Station Availability %）：$$\text{可用度} = \left( 1 - \frac{\text{總累計故障時數}}{\text{總槍數} \times \text{期間總時數}} \right) \times 100\%$$
