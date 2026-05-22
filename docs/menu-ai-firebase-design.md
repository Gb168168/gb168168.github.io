# 📸 菜單照片 → AI 辨識 → Firebase 菜單呈現設計（實作草案）

> 目標：將店家上傳的菜單照片，透過 AI 結構化後寫入 Firestore，並可直接支援 Telegram Inline Keyboard 點餐流程。

## 1) Firestore 結構（建議）

```text
stores/{storeId}/menu/{itemId}
```

每筆 `menu` 文件建議欄位：

- `name: string`
- `category: string`
- `basePrice: number`（可選；有些品項只用 `sizes`）
- `image: string`（可選）
- `sizes: SizeOption[]`（可選）
- `options: OptionGroup[]`（可選）
- `isCombo: boolean`（可選）
- `comboItems: ComboGroup[]`（可選）
- `aiMeta: object`（可選，保存辨識來源與版本）

---

## 2) 三種情境資料模型

### A. 大小份（單選）

```json
{
  "name": "滷肉飯",
  "category": "飯類",
  "sizes": [
    { "label": "小份", "price": 30 },
    { "label": "大份", "price": 50 }
  ]
}
```

### B. 主食選項（必選，單選）

```json
{
  "name": "肉羹",
  "basePrice": 60,
  "options": [
    {
      "groupName": "主食",
      "required": true,
      "maxSelect": 1,
      "choices": [
        { "label": "麵", "extraPrice": 0 },
        { "label": "米粉", "extraPrice": 0 },
        { "label": "冬粉", "extraPrice": 0 },
        { "label": "飯", "extraPrice": 0 },
        { "label": "綜合", "extraPrice": 10 }
      ]
    }
  ]
}
```

### C. 套餐（固定 + 必選/可選）

```json
{
  "name": "A 排骨套餐",
  "basePrice": 150,
  "isCombo": true,
  "comboItems": [
    { "label": "主餐：排骨便當", "fixed": true },
    {
      "label": "附餐飲料",
      "required": true,
      "choices": [
        { "label": "紅茶", "extraPrice": 0 },
        { "label": "綠茶", "extraPrice": 0 },
        { "label": "奶茶", "extraPrice": 10 },
        { "label": "可樂", "extraPrice": 15 }
      ]
    },
    {
      "label": "加購湯品",
      "required": false,
      "choices": [
        { "label": "味噌湯", "extraPrice": 20 },
        { "label": "玉米濃湯", "extraPrice": 25 }
      ]
    }
  ]
}
```

---

## 3) Telegram 呈現規則（Inline Keyboard）

1. **`sizes` 存在時**：先顯示尺寸單選，未選不可加入購物車。
2. **`options[].required=true`**：每個必選群組都要完成選擇。
3. **`maxSelect=1`**：單選（radio）；`maxSelect>1` 用複選（checkbox）。
4. **`comboItems.fixed=true`**：直接顯示 ✅，不提供按鈕。
5. **加價顯示**：`extraPrice>0` 時顯示 `(+${price})`。

> 建議加入購物車前統一驗證：`validateRequiredSelections(item, state)`。

---

## 4) AI 辨識流程（建議）

1. 店家傳菜單照片至 Telegram Bot
2. Bot 取圖後呼叫 Vision 模型
3. 模型輸出結構化 JSON（品名、價格、大小份、選項、套餐）
4. 系統做 schema normalize（欄位清理、價格數字化）
5. 回傳「確認/修改/取消」給店家
6. 確認後寫入 Firestore `stores/{storeId}/menu`

---

## 5) 建議欄位型別（TypeScript）

```ts
export type PriceChoice = {
  label: string;
  extraPrice?: number;
};

export type OptionGroup = {
  groupName: string;
  required: boolean;
  maxSelect: number;
  choices: PriceChoice[];
};

export type ComboGroup = {
  label: string;
  fixed?: boolean;
  required?: boolean;
  maxSelect?: number;
  choices?: PriceChoice[];
};

export type MenuItem = {
  name: string;
  category?: string;
  basePrice?: number;
  image?: string;
  sizes?: Array<{ label: string; price: number }>;
  options?: OptionGroup[];
  isCombo?: boolean;
  comboItems?: ComboGroup[];
  aiMeta?: {
    model?: string;
    sourceImageUrl?: string;
    recognizedAt?: number;
    confidence?: number;
  };
};
```

---

## 6) 實作注意事項

- `basePrice` 與 `sizes` 可並存；若 `sizes` 存在，前台以 size price 為主。
- 金額全部正規化為整數（元），避免浮點誤差。
- AI 解析要保留 `rawText`（可放 `aiMeta`），便於人工校正。
- 寫入 Firestore 前一定要二次確認，避免錯誤上架。

