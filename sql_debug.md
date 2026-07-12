這段 SQL 的目的可以先翻成一句人話：

> 找出 `product` 表與 `vendor` 表中，**vendor_code 相同，但 site_id 不同**的資料。

```sql
SELECT *
FROM product p
JOIN vendor v
  ON p.vendor_code = v.vendor_code
WHERE p.site_id <> v.site_id;
```

## 一行一行解釋

### 1. `FROM product p`

```sql
FROM product p
```

表示主要從 `product` 表開始查詢。

`p` 是 `product` 的別名，後面可以簡寫：

```sql
p.vendor_code
p.site_id
```

而不用每次都寫：

```sql
product.vendor_code
product.site_id
```

---

### 2. `JOIN vendor v`

```sql
JOIN vendor v
```

把 `vendor` 表一起加入查詢。

`v` 是 `vendor` 表的別名。

沒有特別寫 `LEFT JOIN`、`RIGHT JOIN` 時，這裡的 `JOIN` 等同於：

```sql
INNER JOIN
```

代表只有符合連接條件的資料才會被保留。

---

### 3. `ON p.vendor_code = v.vendor_code`

```sql
ON p.vendor_code = v.vendor_code
```

這是兩張表的連接條件：

> 將 `product.vendor_code` 和 `vendor.vendor_code` 相同的資料配對起來。

例如：

### product

| product_id | product_name | site_id | vendor_code |
| ---------: | ------------ | ------- | ----------- |
|          1 | 原子筆          | TAINAN  | V001        |
|          2 | 筆記本          | CHIAYI  | V002        |

### vendor

| site_id | vendor_code | vendor_name |
| ------- | ----------- | ----------- |
| TAINAN  | V001        | 文具公司        |
| TAINAN  | V002        | 紙品公司        |

JOIN 後會得到：

| product_id | product_site_id | vendor_code | vendor_site_id |
| ---------: | --------------- | ----------- | -------------- |
|          1 | TAINAN          | V001        | TAINAN         |
|          2 | CHIAYI          | V002        | TAINAN         |

因為兩筆資料的 `vendor_code` 都能在 `vendor` 表找到對應資料。

---

### 4. `WHERE p.site_id <> v.site_id`

```sql
WHERE p.site_id <> v.site_id
```

`<>` 表示「不等於」。

因此只保留：

```text
product.site_id 不等於 vendor.site_id
```

的資料。

上面的範例最後只會留下：

| product_id | product_site_id | vendor_code | vendor_site_id |
| ---------: | --------------- | ----------- | -------------- |
|          2 | CHIAYI          | V002        | TAINAN         |

這表示：

* 商品屬於嘉義廠區
* 但是這個商品記錄的供應商代碼，對應到了台南廠區的供應商
* 可能存在廠區資料不一致

---

## `SELECT *` 是什麼？

```sql
SELECT *
```

表示把 `product` 和 `vendor` 兩張表的所有欄位都查出來。

但兩張表可能都有：

```text
site_id
vendor_code
```

查詢結果可能出現同名欄位，閱讀上容易混淆。

實際排查時，建議明確選出欄位並重新命名：

```sql
SELECT
    p.product_id,
    p.product_name,
    p.site_id AS product_site_id,
    p.vendor_code,
    v.site_id AS vendor_site_id,
    v.vendor_name
FROM product p
JOIN vendor v
    ON p.vendor_code = v.vendor_code
WHERE p.site_id <> v.site_id;
```

這樣可以直接看出哪一個 `site_id` 是商品的，哪一個是供應商的。

---

## 這段 SQL 有一個重要風險

你之前提到 `vendor` 表的主鍵可能是：

```text
(site_id, vendor_code)
```

也就是 Composite Primary Key。

這代表真正識別一個 Vendor 的方式可能不是只有：

```text
vendor_code
```

而是：

```text
site_id + vendor_code
```

但目前的 JOIN 只用了：

```sql
ON p.vendor_code = v.vendor_code
```

這可能產生「跨廠區錯誤配對」。

例如：

### product

| product_id | site_id | vendor_code |
| ---------: | ------- | ----------- |
|          1 | CHIAYI  | V001        |

### vendor

| site_id | vendor_code | vendor_name |
| ------- | ----------- | ----------- |
| CHIAYI  | V001        | 嘉義供應商       |
| TAINAN  | V001        | 台南供應商       |

執行原本的 JOIN：

```sql
ON p.vendor_code = v.vendor_code
```

商品會同時配對到兩筆 Vendor：

| product_site_id | vendor_code | vendor_site_id |
| --------------- | ----------- | -------------- |
| CHIAYI          | V001        | CHIAYI         |
| CHIAYI          | V001        | TAINAN         |

再套用：

```sql
WHERE p.site_id <> v.site_id
```

第一筆被排除，第二筆留下：

| product_site_id | vendor_code | vendor_site_id |
| --------------- | ----------- | -------------- |
| CHIAYI          | V001        | TAINAN         |

此時查詢會把這筆商品判斷成異常，但實際上它已經存在正確的嘉義 Vendor。

所以這段 SQL 真正查到的是：

> 每個 Product 是否能找到任何一筆「vendor_code 相同、site_id 不同」的 Vendor。

它不完全等同於：

> Product 是否沒有正確的 Vendor。

---

## 更準確地找出「沒有正確 Vendor」的 Product

如果你的目標是找出：

> `product` 裡的 `(site_id, vendor_code)` 在 `vendor` 表中不存在。

建議使用 `NOT EXISTS`：

```sql
SELECT
    p.*
FROM product p
WHERE NOT EXISTS (
    SELECT 1
    FROM vendor v
    WHERE v.site_id = p.site_id
      AND v.vendor_code = p.vendor_code
);
```

它的意思是：

1. 逐筆查看 `product`
2. 到 `vendor` 表尋找相同的 `site_id`
3. 同時尋找相同的 `vendor_code`
4. 如果找不到，就把這筆 Product 列出來

也可以用 `LEFT JOIN`：

```sql
SELECT
    p.*
FROM product p
LEFT JOIN vendor v
    ON p.site_id = v.site_id
   AND p.vendor_code = v.vendor_code
WHERE v.vendor_code IS NULL;
```

## 三種查詢的差異

| 查詢方式                               | 實際用途                        |
| ---------------------------------- | --------------------------- |
| `JOIN vendor_code` 加上 `site_id <>` | 找到同代碼但不同廠區的配對               |
| `NOT EXISTS`                       | 找出不存在正確 Vendor 對應的 Product  |
| `LEFT JOIN ... IS NULL`            | 同樣找出沒有正確 Vendor 對應的 Product |

因此，原本的 SQL 可以作為資料排查線索，但若 `vendor_code` 會在不同廠區重複，就不能直接把查詢結果全部認定為髒資料。
