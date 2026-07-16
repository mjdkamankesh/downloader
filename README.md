# راهنمای توسعه‌دهنده AFTA Business Audit — API یک‌خطی

**نسخه:** Phase 37.5  
**مخاطب:** برنامه‌نویسان API حسابداری، انبار، فروش، خزانه‌داری و سایر ماژول‌های تجاری  
**Namespace:** `ArgessCloud.Api.Infrastructure.Afta`

---

## 1. هدف

برای عملیات معمول روی یک رکورد، برنامه‌نویس فقط یکی از متدهای زیر را فراخوانی می‌کند:

```csharp
AftaAudit.Insert(...);
AftaAudit.Update(...);
AftaAudit.Delete(...);
```

نسخه‌های Async نیز وجود دارند:

```csharp
await AftaAudit.InsertAsync(...);
await AftaAudit.UpdateAsync(...);
await AftaAudit.DeleteAsync(...);
```

Helper به‌صورت مرکزی این کارها را انجام می‌دهد:

- خواندن Policy و Feature Flag از Cache مرکزی؛
- تشخیص روشن یا خاموش بودن AFTA و Business Audit؛
- تشخیص Business Database از Context درخواست؛
- ساخت دیتابیس AFTA با همان معماری موجود؛
- خواندن امن Snapshot قبل و بعد از رکورد؛
- حذف ستون‌های حساس، باینری و حجیم از Snapshot؛
- دریافت User، Company، Financial Year، IP و UserAgent؛
- ساخت Correlation ID و Hash؛
- ثبت نتیجه موفق یا ناموفق؛
- ثبت Sensitive Action در صورت فعال بودن؛
- Fail-open بودن Audit؛
- جلوگیری از Audit تکراری Filter عمومی.

> هیچ ConnectionString جدیدی ساخته نمی‌شود. Helper فقط از `SQLCONFIG` و `SQLCON` استفاده می‌کند.

---

## 2. Namespace لازم

```csharp
using ArgessCloud.Api.Infrastructure.Afta;
```

---

## 3. ساده‌ترین نمونه‌ها

### Insert با ID برگشتی

```csharp
var customerId = AftaAudit.Insert(
    this,
    "dbo.Customer",
    "CustomerId",
    () => CustomerData.Insert(model, CurrentDatabaseName));
```

### Update

```csharp
AftaAudit.Update(
    this,
    "dbo.Customer",
    "CustomerId",
    model.CustomerId,
    () => CustomerData.Update(model, CurrentDatabaseName));
```

### Delete

```csharp
AftaAudit.Delete(
    this,
    "dbo.Customer",
    "CustomerId",
    customerId,
    () => CustomerData.Delete(customerId, CurrentDatabaseName));
```

این سه نمونه برای عملیات معمول کافی‌اند. برنامه‌نویس Policy، دیتابیس AFTA، Snapshot، Hash یا Context را مستقیم مدیریت نمی‌کند.

---

## 4. چرا عملیات اصلی داخل Helper قرار می‌گیرد؟

اشتباه:

```csharp
CustomerData.Update(model, CurrentDatabaseName);
AftaAudit.Update(...);
```

در این حالت وضعیت قبل از تغییر، زمان اجرای واقعی و Exception عملیات در اختیار Framework نیست.

صحیح:

```csharp
AftaAudit.Update(
    this,
    "dbo.Customer",
    "CustomerId",
    model.CustomerId,
    () => CustomerData.Update(model, CurrentDatabaseName));
```

Helper قبل از اجرای Delegate، Snapshot قبلی را می‌خواند؛ سپس عملیات اصلی را اجرا می‌کند و بعد Snapshot جدید را ثبت می‌کند.

---

## 5. Insert

### 5.1 Insert با شناسه برگشتی

برای متدهایی که ID جدید را برمی‌گردانند:

```csharp
var id = AftaAudit.Insert(
    this,
    "sales.Customer",
    "CustomerId",
    () => CustomerData.Insert(model, CurrentDatabaseName));
```

مقدار برگشتی Delegate به‌عنوان شناسه رکورد استفاده می‌شود و Snapshot بعد از Insert از همان رکورد خوانده می‌شود.

نوع ID می‌تواند `int`، `long`، `Guid`، `string` یا سایر نوع‌های قابل ارسال به SQL Server باشد.

### 5.2 Insert با شناسه از قبل مشخص

برای Guid یا کلیدی که قبل از Insert ساخته شده است:

```csharp
var id = Guid.NewGuid();

AftaAudit.Insert(
    this,
    "dbo.Project",
    "ProjectId",
    id,
    () => ProjectData.Insert(id, model, CurrentDatabaseName));
```

### 5.3 Insert بدون مقدار برگشتی

اگر متد اصلی `void` است ولی ID از قبل مشخص است، از همان overload بالا استفاده کنید.

---

## 6. Update

```csharp
AftaAudit.Update(
    this,
    "inventory.WarehouseReceipt",
    "ReceiptId",
    model.ReceiptId,
    () => WarehouseReceiptData.Update(model, CurrentDatabaseName));
```

ترتیب داخلی:

1. Policy مرکزی بررسی می‌شود.
2. فقط در صورت فعال بودن Snapshot، وضعیت قبل خوانده می‌شود.
3. Update اصلی اجرا می‌شود.
4. فقط در صورت موفقیت، وضعیت بعد خوانده می‌شود.
5. Audit موفق ثبت می‌شود.
6. در صورت Exception، Audit ناموفق ثبت و همان Exception دوباره پرتاب می‌شود.

### Update دارای مقدار برگشتی

```csharp
var affectedRows = AftaAudit.Update(
    this,
    "dbo.Customer",
    "CustomerId",
    model.CustomerId,
    () => CustomerData.UpdateAndReturnAffectedRows(model, CurrentDatabaseName));
```

مقدار برگشتی بدون تغییر به Controller بازگردانده می‌شود.

---

## 7. Delete

```csharp
AftaAudit.Delete(
    this,
    "inventory.WarehouseReceipt",
    "ReceiptId",
    receiptId,
    () => WarehouseReceiptData.Delete(receiptId, CurrentDatabaseName));
```

در Delete:

- Snapshot قبل از حذف خوانده می‌شود؛
- عملیات حذف اجرا می‌شود؛
- پس از موفقیت، Marker حذف شامل نام جدول، ستون کلید و ID ثبت می‌شود؛
- ریسک پیش‌فرض `High` و Sensitive Level پیش‌فرض `3` است.

### Delete دارای مقدار برگشتی

```csharp
var deletedCount = AftaAudit.Delete(
    this,
    "dbo.Customer",
    "CustomerId",
    customerId,
    () => CustomerData.DeleteAndReturnCount(customerId, CurrentDatabaseName));
```

---

## 8. نسخه‌های Async

### Insert Async

```csharp
var id = await AftaAudit.InsertAsync(
    this,
    "dbo.Customer",
    "CustomerId",
    () => CustomerData.InsertAsync(model, CurrentDatabaseName));
```

### Update Async

```csharp
await AftaAudit.UpdateAsync(
    this,
    "dbo.Customer",
    "CustomerId",
    model.CustomerId,
    () => CustomerData.UpdateAsync(model, CurrentDatabaseName));
```

### Delete Async

```csharp
await AftaAudit.DeleteAsync(
    this,
    "dbo.Customer",
    "CustomerId",
    customerId,
    () => CustomerData.DeleteAsync(customerId, CurrentDatabaseName));
```

---

## 9. تنظیمات اختیاری

برای عملیات معمول نیازی به ساخت Options نیست. در موارد حساس یا خاص:

```csharp
AftaAudit.Update(
    this,
    "dbo.Customer",
    "CustomerId",
    model.CustomerId,
    () => CustomerData.Update(model, CurrentDatabaseName),
    new AftaAuditOptions
    {
        OperationKey = "Sales.Customer.Update",
        OperationTitle = "ویرایش مشتری",
        EntityType = "Customer",
        Reason = model.ChangeReason,
        RiskLevel = "Medium",
        SensitiveLevel = 2,
        IsSensitiveAction = true,
        ExcludedColumns = new[] { "ProfileImage", "InternalLargeText" }
    });
```

### فیلدهای AftaAuditOptions

| فیلد | کاربرد |
|---|---|
| `OperationKey` | کلید پایدار گزارش با الگوی `Module.Entity.Operation` |
| `OperationTitle` | عنوان فارسی قابل فهم |
| `EntityType` | نام موجودیت؛ در صورت عدم تعیین از نام جدول ساخته می‌شود |
| `Reason` | دلیل تجاری عملیات |
| `RiskLevel` | `Low`، `Medium`، `High` یا `Critical` |
| `SensitiveLevel` | عدد 0 تا 5 |
| `IsSensitiveAction` | ثبت یا عدم ثبت Sensitive Action |
| `SkipAutomaticApiAudit` | جلوگیری از Audit تکراری؛ معمولاً تغییر نکند |
| `ExcludedColumns` | ستون‌های اضافه‌ای که نباید وارد Snapshot شوند |
| `MaxColumns` | حداکثر تعداد ستون‌ها؛ پیش‌فرض 128 |
| `MaxValueLength` | حداکثر طول هر متن؛ پیش‌فرض 4000 |
| `RecordDeleteMarker` | ثبت Marker بعد از Delete؛ پیش‌فرض true |

---

## 10. رفتار Snapshot Reader

Helper نام جدول را به یکی از شکل‌های زیر می‌پذیرد:

```text
Customer
dbo.Customer
sales.Invoice
```

در صورت حذف Schema، مقدار `dbo` استفاده می‌شود.

قواعد امنیتی:

- نام Schema، جدول و ستون فقط از حروف، عدد و `_` تشکیل می‌شود؛
- نام باید با حرف یا `_` شروع شود؛
- مقدار ID همیشه SQL Parameter است؛
- جدول و ستون با Metadata واقعی دیتابیس کنترل می‌شوند؛
- نام جدول نباید از ورودی مستقیم کاربر گرفته شود؛
- فقط Business Database موجود در Context باز می‌شود؛
- Helper اجازه دریافت DatabaseName جداگانه ندارد.

### ستون‌های حذف‌شده به‌صورت پیش‌فرض

ستون‌هایی با نام یا مفهوم زیر وارد Snapshot نمی‌شوند:

- Password و PasswordHash؛
- Token و RefreshToken؛
- Secret، PrivateKey و ApiKey؛
- ConnectionString و Cookie؛
- اطلاعات حساس کارت؛
- انواع `binary`، `varbinary` و `image`؛
- `xml`، `text` و `ntext`؛
- ستون‌های `max` یا بسیار حجیم.

Masking مرکزی AFTA نیز بعد از خواندن Snapshot همچنان اجرا می‌شود.

---

## 11. Performance و حالت خاموش

وقتی AFTA یا Business Audit خاموش باشد:

- فقط Snapshot Policy از Cache مرکزی خوانده می‌شود؛
- هیچ Query برای Before/After اجرا نمی‌شود؛
- هیچ Hash ساخته نمی‌شود؛
- هیچ Write به دیتابیس AFTA انجام نمی‌شود؛
- فقط Delegate اصلی اجرا می‌شود.

وقتی Audit روشن ولی ثبت Snapshot خاموش باشد، عملیات Audit پایه ثبت می‌شود اما Queryهای رکورد اجرا نمی‌شوند.

---

## 12. Transaction

اگر عملیات اصلی Transaction داخلی دارد، کل عملیات و Commit داخل Delegate باشد:

```csharp
AftaAudit.Update(
    this,
    "dbo.Invoice",
    "InvoiceId",
    invoiceId,
    () => InvoiceService.UpdateWithTransaction(model, CurrentDatabaseName));
```

Audit موفق فقط بعد از بازگشت موفق Delegate ثبت می‌شود.

Executor جایگزین Transaction، Validation یا Business Rule نیست.

---

## 13. عملیات پیچیده

Helper یک‌خطی برای عملیات معمول روی یک رکورد طراحی شده است. در این موارد از `AftaBusinessAuditExecutor` کامل استفاده کنید:

- کلید مرکب؛
- عملیات روی چند جدول؛
- Header/Detail یا Aggregate پیچیده؛
- عملیات Batch؛
- Post، Confirm، Close، Cancel یا Restore؛
- Snapshot سفارشی؛
- عملیات بدون یک رکورد مشخص.

نمونه مسیر پیشرفته:

```csharp
AftaBusinessAuditExecutor.ExecuteUpdate(this, command, () => ComplexBusinessOperation());
```

---

## 14. موارد ممنوع

- تغییر `ArgessAcc.Model`، `ArgessAcc.DAL` یا `ArgessAcc.BLL`؛
- ارسال DatabaseName به Helper؛
- گرفتن نام جدول یا ستون از Request کاربر؛
- فراخوانی مستقیم جدول‌ها یا Stored Procedureهای AFTA؛
- ساخت ConnectionString سوم؛
- اجرای عملیات اصلی قبل یا بعد از Helper؛
- Trigger عمومی برای Business Audit؛
- ActionFilter عمومی برای Snapshot تجاری؛
- Snapshot دستی برای عملیات معمول یک‌جدولی؛
- بلعیدن Exception عملیات اصلی.

---

## 15. Checklist برنامه‌نویس

- [ ] عملیات اصلی داخل `AftaAudit.Insert/Update/Delete` قرار دارد.
- [ ] نام جدول واقعی و دارای Schema صحیح است.
- [ ] نام ستون کلید واقعی است.
- [ ] مقدار ID مربوط به همان رکورد است.
- [ ] نام جدول و ستون ثابت کد هستند و از Request نمی‌آیند.
- [ ] برای عملیات حساس، Options مناسب ثبت شده است.
- [ ] ستون‌های حجیم یا محرمانه اختصاصی در `ExcludedColumns` آمده‌اند.
- [ ] Validation ناموفق قبل از ورود به Helper متوقف می‌شود.
- [ ] حالت Audit روشن و خاموش تست شده است.
- [ ] پروژه‌های `ArgessAcc.*` تغییر نکرده‌اند.

---

## 16. الگوهای آماده Copy/Paste

### Insert

```csharp
var id = AftaAudit.Insert(
    this,
    "dbo.EntityTable",
    "EntityId",
    () => EntityData.Insert(model, CurrentDatabaseName));
```

### Update

```csharp
AftaAudit.Update(
    this,
    "dbo.EntityTable",
    "EntityId",
    model.EntityId,
    () => EntityData.Update(model, CurrentDatabaseName));
```

### Delete

```csharp
AftaAudit.Delete(
    this,
    "dbo.EntityTable",
    "EntityId",
    entityId,
    () => EntityData.Delete(entityId, CurrentDatabaseName));
```

### Update حساس

```csharp
AftaAudit.Update(
    this,
    "treasury.BankAccount",
    "BankAccountId",
    model.BankAccountId,
    () => BankAccountData.Update(model, CurrentDatabaseName),
    new AftaAuditOptions
    {
        OperationKey = "Treasury.BankAccount.Update",
        OperationTitle = "ویرایش حساب بانکی",
        Reason = model.ChangeReason,
        RiskLevel = "High",
        SensitiveLevel = 4,
        ExcludedColumns = new[] { "EncryptedAccountData" }
    });
```

---

## 17. جمع‌بندی معماری

```text
Business API Controller
        ↓
AftaAudit.Insert / Update / Delete
        ↓
AftaBusinessAuditExecutor
        ↓
Central AftaPolicyProvider / Policy Cache
        ↓
BusinessAftaServiceFactory
        ↓
Business Audit Log / Sensitive Action
```

برای عملیات معمول، مسئولیت برنامه‌نویس فقط معرفی جدول، ستون کلید، ID و Delegate عملیات اصلی است. تمام Policy checkها، Context، Snapshot، Masking، Hash، Correlation، ثبت موفق/ناموفق و Fail-open داخل Framework باقی می‌ماند.
