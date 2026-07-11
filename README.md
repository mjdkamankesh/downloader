# راهنمای توسعه‌دهنده Business Audit در ArgessCloud API

**نسخه:** Phase 37.4  
**مخاطب:** برنامه‌نویسان ماژول‌های حسابداری، انبار، فروش، خزانه‌داری و سایر عملیات تجاری  
**محل SDK:** `ArgessCloud.Api/Infrastructure/Afta`

---

## 1. هدف

برای ثبت Audit عملیات تجاری، نباید در پروژه‌های `ArgessAcc.Model`، `ArgessAcc.DAL` و `ArgessAcc.BLL` تغییری ایجاد شود. برنامه‌نویس فقط در Action نهایی API و دقیقاً در نقطه‌ای که عملیات واقعی Insert، Update یا Delete انجام می‌شود، متد استاندارد AFTA را فراخوانی می‌کند.

Framework به‌صورت مرکزی این موارد را مدیریت می‌کند:

- خواندن Feature Flag و Security Setting از Policy Cache مرکزی؛
- تشخیص روشن یا خاموش بودن AFTA و Business Audit؛
- تشخیص Business Database و ساخت نام دیتابیس AFTA با الگوی `BusinessDatabaseName + DatabaseSuffix`؛
- دریافت User، Company، Financial Year، IP، UserAgent و اطلاعات Request؛
- ایجاد Correlation ID؛
- ثبت زمان اجرای عملیات و نتیجه موفق یا ناموفق؛
- دریافت Snapshot قبل و بعد، فقط در صورت فعال بودن تنظیم مربوط؛
- Mask کردن اطلاعات حساس Snapshot؛
- ساخت SHA-256 برای کنترل تمامیت؛
- ثبت Business Audit Log و Sensitive Action؛
- جلوگیری از ثبت Audit تکراری توسط Audit Filter عمومی؛
- Fail-open بودن Audit؛ یعنی خطای سامانه Audit نباید عملیات اصلی حسابداری یا تجاری را متوقف کند.

> برنامه‌نویس نباید مستقیماً جدول‌های AFTA، Stored Procedureهای Audit، Policy Provider یا ConnectionStringها را صدا بزند.

---

## 2. Namespaceهای لازم

```csharp
using ArgessCloud.Api.Infrastructure.Afta;
```

برای Snapshotهای JSON معمولاً این Namespace نیز لازم است:

```csharp
using Newtonsoft.Json;
```

---

## 3. متدهای استاندارد SDK

برای عملیات Sync:

```csharp
AftaBusinessAuditExecutor.ExecuteInsert(...);
AftaBusinessAuditExecutor.ExecuteUpdate(...);
AftaBusinessAuditExecutor.ExecuteDelete(...);
```

برای عملیات Async:

```csharp
await AftaBusinessAuditExecutor.ExecuteInsertAsync(...);
await AftaBusinessAuditExecutor.ExecuteUpdateAsync(...);
await AftaBusinessAuditExecutor.ExecuteDeleteAsync(...);
```

هر متد در دو حالت قابل استفاده است:

1. عملیات دارای مقدار بازگشتی، مانند شناسه رکورد Insert‌شده؛
2. عملیات بدون مقدار بازگشتی، مانند یک متد `void` یا `Task`.

متدهای عمومی `Execute` و `ExecuteAsync` برای عملیات خاص مانند Post، Confirm، Close یا Cancel وجود دارند؛ برای Insert/Update/Delete همیشه از متد نام‌دار همان عملیات استفاده شود.

---

## 4. قرارداد اصلی استفاده

ساختار صحیح همیشه به این صورت است:

```csharp
var command = new AftaBusinessAuditCommand
{
    OperationKey = "Module.Entity.Operation",
    OperationTitle = "عنوان فارسی عملیات",
    EntityType = "EntityName",
    EntityId = entityId,
    Reason = "علت انجام عملیات",
    BeforeSnapshot = () => LoadBeforeSnapshot(),
    AfterSnapshot = () => LoadAfterSnapshot()
};

AftaBusinessAuditExecutor.ExecuteUpdate(
    this,
    command,
    () => BusinessMethod());
```

سه بخش اصلی عبارت‌اند از:

- `this`: Controller فعلی API؛
- `command`: مشخصات Audit؛
- `action`: همان عملیات اصلی برنامه که باید اجرا شود.

عملیات اصلی را بیرون Executor اجرا نکنید. در غیر این صورت مدت اجرا، نتیجه، خطا و Snapshotها با عملیات واقعی هم‌بسته نخواهند بود.

---

## 5. نمونه کامل Insert با مقدار بازگشتی

```csharp
[HttpPost]
[Route("insert")]
public IHttpActionResult Insert(CustomerModel model)
{
    var command = new AftaBusinessAuditCommand
    {
        OperationKey = "Sales.Customer.Insert",
        OperationTitle = "ثبت مشتری",
        EntityType = "Customer",
        Reason = "ثبت مشتری از API فروش",
        RiskLevel = "Medium",
        SensitiveLevel = 2,
        IsSensitiveAction = true
    };

    var insertedId = AftaBusinessAuditExecutor.ExecuteInsert(
        this,
        command,
        () =>
        {
            var id = CustomerData.Insert(model, CurrentDatabaseName);

            command.EntityId = id.ToString();
            command.AfterSnapshot = () => BuildCustomerAuditSnapshot(id);

            return id;
        });

    return Ok(new ApiResult<long>
    {
        IsSuccess = true,
        Message = "مشتری با موفقیت ثبت شد",
        Data = insertedId
    });
}
```

نکات Insert:

- قبل از Insert معمولاً `BeforeSnapshot` وجود ندارد.
- بعد از دریافت ID، مقدار `command.EntityId` را تنظیم کنید.
- `AfterSnapshot` را بعد از مشخص شدن ID تنظیم کنید تا Framework رکورد واقعی ثبت‌شده را بخواند.
- اگر متد Insert مقدار برنمی‌گرداند، از overload بدون خروجی استفاده کنید.

---

## 6. نمونه Insert بدون مقدار بازگشتی

```csharp
AftaBusinessAuditExecutor.ExecuteInsert(
    this,
    new AftaBusinessAuditCommand
    {
        OperationKey = "Inventory.StockOpening.Insert",
        OperationTitle = "ثبت موجودی اول دوره",
        EntityType = "StockOpening",
        EntityId = model.Id.ToString(),
        Reason = "ثبت موجودی اول دوره انبار"
    },
    () => StockOpeningData.Insert(model, CurrentDatabaseName));
```

---

## 7. نمونه کامل Update

```csharp
[HttpPost]
[Route("update")]
public IHttpActionResult Update(CustomerModel model)
{
    var entityId = model.CustomerId.ToString();

    AftaBusinessAuditExecutor.ExecuteUpdate(
        this,
        new AftaBusinessAuditCommand
        {
            OperationKey = "Sales.Customer.Update",
            OperationTitle = "ویرایش مشتری",
            EntityType = "Customer",
            EntityId = entityId,
            Reason = "ویرایش اطلاعات مشتری از API فروش",
            RiskLevel = "Medium",
            SensitiveLevel = 2,
            IsSensitiveAction = true,
            BeforeSnapshot = () => BuildCustomerAuditSnapshot(model.CustomerId),
            AfterSnapshot = () => BuildCustomerAuditSnapshot(model.CustomerId)
        },
        () => CustomerData.Update(model, CurrentDatabaseName));

    return Ok(new ApiResult<object>
    {
        IsSuccess = true,
        Message = "مشتری با موفقیت ویرایش شد"
    });
}
```

ترتیب اجرا توسط Framework:

1. Policy مرکزی بررسی می‌شود.
2. در صورت فعال بودن Snapshot، `BeforeSnapshot` خوانده می‌شود.
3. متد Update واقعی اجرا می‌شود.
4. فقط پس از موفقیت، `AfterSnapshot` خوانده می‌شود.
5. Audit موفق ثبت می‌شود.
6. اگر Update Exception بدهد، Audit ناموفق ثبت و همان Exception دوباره پرتاب می‌شود.

---

## 8. نمونه کامل Delete

```csharp
[HttpPost]
[Route("delete")]
public IHttpActionResult Delete(Guid id, Guid userId)
{
    AftaBusinessAuditExecutor.ExecuteDelete(
        this,
        new AftaBusinessAuditCommand
        {
            OperationKey = "Inventory.WarehouseReceipt.Delete",
            OperationTitle = "حذف رسید انبار",
            EntityType = "WarehouseReceipt",
            EntityId = id.ToString(),
            Reason = "حذف رسید انبار توسط کاربر",
            RiskLevel = "High",
            SensitiveLevel = 4,
            IsSensitiveAction = true,
            BeforeSnapshot = () => BuildReceiptAuditSnapshot(id),
            AfterSnapshot = () => JsonConvert.SerializeObject(new
            {
                Deleted = true,
                ReceiptId = id,
                DeletedBy = userId
            })
        },
        () => WarehouseReceiptData.Delete(id, userId, CurrentDatabaseName));

    return Ok(new ApiResult<object>
    {
        IsSuccess = true,
        Message = "رسید انبار با موفقیت حذف شد"
    });
}
```

برای Delete، SDK در صورت تعیین نشدن مقدار مناسب، سطح ریسک را `High` و `SensitiveLevel` را حداقل `3` در نظر می‌گیرد. بااین‌حال برای عملیات بسیار حساس، مقدار صریح مانند `4` یا `5` ثبت شود.

---

## 9. نمونه Async

```csharp
[HttpPost]
[Route("update-async")]
public async Task<IHttpActionResult> UpdateAsync(CustomerModel model)
{
    await AftaBusinessAuditExecutor.ExecuteUpdateAsync(
        this,
        new AftaBusinessAuditCommand
        {
            OperationKey = "Sales.Customer.Update",
            OperationTitle = "ویرایش مشتری",
            EntityType = "Customer",
            EntityId = model.CustomerId.ToString(),
            Reason = "ویرایش Async مشتری",
            BeforeSnapshot = () => BuildCustomerAuditSnapshot(model.CustomerId),
            AfterSnapshot = () => BuildCustomerAuditSnapshot(model.CustomerId)
        },
        () => CustomerData.UpdateAsync(model, CurrentDatabaseName));

    return Ok();
}
```

Callbackهای Snapshot در نسخه فعلی Sync هستند. داخل آن‌ها Query کوتاه و مستقیم اجرا کنید و عملیات طولانی یا شبکه‌ای قرار ندهید.

---

## 10. تعریف فیلدهای AftaBusinessAuditCommand

| فیلد | اجباری | توضیح |
|---|---:|---|
| `OperationKey` | بله | کلید یکتا با الگوی `Module.Entity.Operation`؛ نمونه: `Accounting.Sanad.Update` |
| `OperationTitle` | بله | عنوان فارسی قابل فهم برای مدیر و Auditor |
| `EntityType` | بله | نام موجودیت تجاری؛ نمونه: `Sanad`، `Invoice`، `WarehouseReceipt` |
| `EntityId` | توصیه‌شده | شناسه رکورد؛ در Insert می‌تواند پس از عملیات تنظیم شود |
| `Reason` | برای عملیات حساس بله | دلیل تجاری یا منبع انجام عملیات؛ از متن آزاد کاربر بدون کنترل استفاده نشود |
| `RiskLevel` | خیر | `Low`، `Medium`، `High` یا `Critical`؛ پیش‌فرض `Medium` |
| `SensitiveLevel` | خیر | عدد 0 تا 5؛ پیش‌فرض 2؛ Delete حداقل 3 |
| `IsSensitiveAction` | خیر | تعیین می‌کند رکورد Sensitive Action نیز ثبت شود؛ پیش‌فرض `true` |
| `SkipAutomaticApiAudit` | خیر | مانع Audit تکراری Filter عمومی می‌شود؛ پیش‌فرض `true` و معمولاً تغییر نکند |
| `BeforeSnapshot` | خیر | تابعی برای خواندن وضعیت قبل از عملیات |
| `AfterSnapshot` | خیر | تابعی برای خواندن وضعیت بعد از عملیات |
| `AfterSnapshotFromResult` | خیر | ساخت Snapshot بعد از عملیات با استفاده از نتیجه برگشتی |
| `SuccessStatusCode` | خیر | Status Code موفق ثبت‌شده در Audit؛ پیش‌فرض `200 OK` |

---

## 11. استاندارد OperationKey

فرمت ثابت:

```text
Module.Entity.Operation
```

نمونه‌ها:

```text
Accounting.Sanad.Insert
Accounting.Sanad.Update
Accounting.Sanad.Delete
Inventory.WarehouseReceipt.Insert
Sales.Invoice.Confirm
Treasury.Payment.Cancel
```

قواعد:

- فقط نام فنی پایدار و انگلیسی استفاده شود.
- نام Controller یا Route متغیر را به‌عنوان کلید قرار ندهید.
- برای یک عملیات واحد، در نقاط مختلف برنامه کلیدهای متفاوت نسازید.
- تغییر عنوان فارسی مجاز است؛ تغییر بی‌دلیل OperationKey باعث شکستن گزارش‌های تاریخی می‌شود.

---

## 12. ساخت Snapshot امن

Snapshot باید کوچک، قابل فهم و محدود به اطلاعات لازم Audit باشد.

```csharp
private string BuildCustomerAuditSnapshot(long customerId)
{
    var customer = CustomerData.Select(customerId, CurrentDatabaseName);
    if (customer == null)
    {
        return null;
    }

    return JsonConvert.SerializeObject(new
    {
        customer.CustomerId,
        customer.Code,
        customer.Title,
        customer.IsActive,
        customer.ModifiedDate
    });
}
```

موارد ممنوع در Snapshot:

- Password، Token، Cookie، ConnectionString و Secret؛
- اطلاعات کامل کارت بانکی یا داده‌های محرمانه غیرضروری؛
- فایل، تصویر یا Base64؛
- لیست‌های بسیار بزرگ؛
- کل Request یا کل Model بدون انتخاب فیلدهای لازم؛
- داده‌ای که بازیابی آن Query بسیار سنگین دارد.

Framework متن Snapshot را Mask می‌کند، اما این قابلیت جایگزین انتخاب صحیح داده توسط برنامه‌نویس نیست.

### Snapshot و Diff

Framework وضعیت قبل و بعد را در یک Correlation واحد ثبت می‌کند؛ بنابراین تغییرات از مقایسه `BeforeSnapshot` و `AfterSnapshot` قابل استخراج است. در نسخه فعلی، Diff ساخت‌یافته و فیلدبه‌فیلد در ستون مستقل ذخیره نمی‌شود. برای حفظ پایداری و جلوگیری از تغییر Schema در Phase 37.4، منبع معتبر تغییر همان زوج Snapshot و Hashهای آن‌ها است.

---

## 13. رفتار Performance و Zero-cost

وقتی AFTA یا Business Audit خاموش باشد:

- فقط Policy Snapshot از Cache مرکزی خوانده می‌شود؛
- Snapshotهای Before/After اجرا نمی‌شوند؛
- SHA-256 ساخته نمی‌شود؛
- اتصال یا Write به دیتابیس AFTA انجام نمی‌شود؛
- فقط عملیات اصلی برنامه اجرا می‌شود.

وقتی Audit روشن ولی `Audit.RecordMutationSnapshots=false` باشد:

- عملیات و Audit پایه ثبت می‌شوند؛
- Callbackهای Snapshot اجرا نمی‌شوند؛
- هزینه Queryهای Before/After حذف می‌شود.

برنامه‌نویس نباید قبل از ورود به Executor، Snapshot را محاسبه کند. این اشتباه باعث می‌شود حتی در حالت خاموش بودن Audit نیز هزینه Query پرداخت شود.

اشتباه:

```csharp
var before = BuildSnapshot(id);
command.BeforeSnapshot = () => before;
```

صحیح:

```csharp
command.BeforeSnapshot = () => BuildSnapshot(id);
```

---

## 14. مدیریت خطا

Framework دو اصل دارد:

1. Exception عملیات اصلی مخفی نمی‌شود و پس از ثبت Audit ناموفق دوباره پرتاب می‌شود.
2. Exception داخلی Audit، Policy یا دیتابیس AFTA عملیات اصلی را متوقف نمی‌کند.

بنابراین Controller همچنان باید Validation و Exception handling معمول برنامه را حفظ کند. Executor جایگزین Validation، Transaction یا Business Rule نیست.

اگر عملیات اصلی داخل Transaction انجام می‌شود، کل Commit واقعی را داخل `action` قرار دهید. Audit باید فقط پس از موفقیت متد اصلی، نتیجه موفق ثبت کند.

---

## 15. مواردی که نباید انجام شود

- تغییر مستقیم `ArgessAcc.Model`، `ArgessAcc.DAL` یا `ArgessAcc.BLL` برای Audit؛
- استفاده از Trigger عمومی دیتابیس؛
- استفاده از ActionFilter عمومی برای Snapshot عملیات تجاری؛
- فراخوانی مستقیم `BusinessAftaServiceFactory` از Controllerهای تجاری؛
- خواندن مستقیم Feature Flag یا Security Setting در Controller؛
- ساخت ConnectionString سوم برای AFTA؛
- ثبت Audit قبل از موفقیت عملیات؛
- اجرای عملیات اصلی بیرون Executor؛
- استفاده از یک Snapshot سنگین برای کل Aggregate؛
- بلعیدن Exception عملیات اصلی؛
- ثبت اطلاعات حساس خام در `Reason` یا Snapshot.

---

## 16. Checklist قبل از Commit

- [ ] عملیات واقعی داخل یکی از متدهای `ExecuteInsert/Update/Delete` قرار دارد.
- [ ] `OperationKey` از الگوی ثابت پیروی می‌کند.
- [ ] `OperationTitle` فارسی و قابل فهم است.
- [ ] `EntityType` و در صورت امکان `EntityId` ثبت شده‌اند.
- [ ] برای Update و Delete، Snapshot قبل از عملیات تعریف شده است.
- [ ] Snapshot بعد از موفقیت عملیات خوانده می‌شود.
- [ ] Snapshot کوچک و فاقد Secret است.
- [ ] برای عملیات حساس، `Reason`، `RiskLevel` و `SensitiveLevel` مناسب تعیین شده‌اند.
- [ ] عملیات Cancel‌شده یا Validation ناموفق وارد Executor نمی‌شود.
- [ ] Controller مستقیماً Policy، Cache یا دیتابیس AFTA را صدا نمی‌زند.
- [ ] پروژه‌های `ArgessAcc.*` تغییر نکرده‌اند.
- [ ] حالت Audit روشن و خاموش هر دو تست شده‌اند.
- [ ] در صورت Exception عملیات اصلی، همان Exception به لایه بالاتر می‌رسد.

---

## 17. الگوی آماده برای Copy/Paste

```csharp
var command = new AftaBusinessAuditCommand
{
    OperationKey = "Module.Entity.Update",
    OperationTitle = "ویرایش موجودیت",
    EntityType = "Entity",
    EntityId = id.ToString(),
    Reason = "ویرایش موجودیت از API",
    RiskLevel = "Medium",
    SensitiveLevel = 2,
    IsSensitiveAction = true,
    BeforeSnapshot = () => BuildAuditSnapshot(id),
    AfterSnapshot = () => BuildAuditSnapshot(id)
};

AftaBusinessAuditExecutor.ExecuteUpdate(
    this,
    command,
    () => EntityData.Update(model, CurrentDatabaseName));
```

---

## 18. مسیر نمونه واقعی در پروژه

نمونه عملی Insert، Update و Delete سند حسابداری در فایل زیر موجود است:

```text
ArgessCloud.Api/Controllers/ArgessAccControllers/SanadController.cs
```

این فایل فقط نمونه مصرف SDK است. توسعه Audit برای سایر ماژول‌ها باید در Controllerهای API همان ماژول و بدون دستکاری پروژه‌های `ArgessAcc.*` انجام شود.

---

## 19. جمع‌بندی معماری

مسیر استاندارد نهایی:

```text
Business API Controller
        ↓
AftaBusinessAuditExecutor
        ↓
Central AftaPolicyProvider / Policy Cache
        ↓
BusinessAftaServiceFactory
        ↓
BusinessDatabaseName + AFTA Suffix
        ↓
Business Audit Log / Sensitive Action
```

تنها مسئولیت برنامه‌نویس ماژول تجاری این است که عملیات اصلی، مشخصات Operation و Snapshotهای کوچک و امن را به Executor بدهد. تمام تصمیم‌گیری‌های Policy، Context، Correlation، Masking، Hash، ثبت موفق/ناموفق و Fail-open داخل Framework انجام می‌شود.

