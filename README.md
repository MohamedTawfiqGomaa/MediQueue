# شرح مشروع MediQueue بالكامل

## ما هو المشروع؟

MediQueue هو نظام إدارة عيادات طبية على الويب، مبني بـ **ASP.NET Core MVC**.
بيحل مشكلة الزحام في العيادات عن طريق:
- حجز مواعيد أونلاين
- تتبع الدور (Queue) لحظة بلحظة
- لوحة تحكم للطبيب والأدمن

---

## هيكل المشروع

```
MediQueue/
├── Models/       ← جداول قاعدة البيانات
├── BL/           ← منطق العمل (Business Logic)
├── Controllers/  ← المحركات (بتستقبل الطلبات)
├── ViewModel/    ← بيانات النماذج (Forms)
├── Views/        ← الصفحات (cshtml)
├── wwwroot/      ← CSS, JS, Images
└── Program.cs    ← نقطة البداية
```

---

## أولاً: Models — قاعدة البيانات

كل class هنا بيتحول لـ Table في SQL Server.

### الجداول (Entities)

| الملف | يمثل | أهم الحقول |
|---|---|---|
| `User.cs` | كل مستخدم (مريض/طبيب/أدمن) | FullName, Specialty, AvailableSlots, Rating |
| `Clinic.cs` | العيادة | Name, Address, PhoneNumber |
| `Appointment.cs` | الحجز | PatientID, DoctorID, Date, Time, Status |
| `Queue.cs` | الدور | Position, EstimatedTime, IsActive |
| `AppointmentStatus.cs` | حالة الموعد | Booked, Confirmed, Completed, Cancelled |

### User.cs — أهم ملف في المشروع

```csharp
public class User : IdentityUser   // ← وارث من نظام التأمين
{
    public string FullName { get; set; }
    // حقول خاصة بالطبيب بس
    public string? Specialty { get; set; }
    public int? ClinicID { get; set; }
    public decimal? Rating { get; set; }
    public decimal? PricePerSession { get; set; }
    public string? AvailableSlots { get; set; }  // "09:00,10:30,13:00"
}
```

الفكرة الذكية هنا إن **جدول واحد بيخدم 3 أدوار مختلفة**:

المريض → FullName + Email بس
الطبيب → + Specialty + Rating + AvailableSlots
الأدمن → نفس User بس Role = "Admin"

### Appointment.cs — قلب النظام

```csharp
public class Appointment
{
    public string PatientID { get; set; }   // FK → User
    public string DoctorID { get; set; }    // FK → User
    public DateTime AppointmentDate { get; set; }
    public TimeSpan AppointmentTime { get; set; }
    public AppointmentStatus Status { get; set; } = AppointmentStatus.Booked;
    public Queue Queue { get; set; }        // One-to-One مع الدور
}
```

### AppointmentStatus — دورة حياة الموعد

```
Booked → Confirmed → Completed
             ↓
          Cancelled
```

| الحالة | المعنى |
|---|---|
| Booked | حُجز لكن لسه ما اتأكدش |
| Confirmed | مؤكد |
| Completed | انتهى |
| Cancelled | أُلغي |

### Queue.cs — نظام الدور

```csharp
public class Queue
{
    public int AppointmentID { get; set; }  // One-to-One مع الموعد
    public string DoctorID { get; set; }
    public int Position { get; set; }       // رقم الدور (1, 2, 3...)
    public DateTime EstimatedTime { get; set; }
    public bool IsActive { get; set; } = true;
}
```

### العلاقات بين الجداول

```
Clinic ──────< User (Doctor)
                    │
           One-to-Many (DoctorAppointments)
                    │
                    ▼
User (Patient) >── Appointment ──── Queue
                        │
                 AppointmentStatus
          (Booked/Confirmed/Completed/Cancelled)
```

### MediQueueContext.cs — حلقة الوصل مع DB

```csharp
public class MediQueueContext : IdentityDbContext<User>
{
    public DbSet<Clinic> Clinics { get; set; }
    public DbSet<Appointment> Appointments { get; set; }
    public DbSet<Queue> Queues { get; set; }
}
```

`IdentityDbContext<User>` معناها في جداول تانية تلقائية (AspNetUsers, AspNetRoles...).

### OnModelCreating — تجنب الـ Cascade Delete

```csharp
// لو مسحنا طبيب — الموعد يتشال منه بس ما يتمسحش
.OnDelete(DeleteBehavior.Restrict);

// Appointment و Queue علاقة One-to-One
modelBuilder.Entity<Appointment>()
    .HasOne(a => a.Queue)
    .WithOne(q => q.Appointment)
    .HasForeignKey<Queue>(q => q.AppointmentID);
```

---

## ثانياً: DbInitializer.cs — البيانات التجريبية

بيشتغل تلقائياً لما المشروع يبدأ أول مرة.

```
Program.cs يبدأ
      ↓
SeedRoles()              → بيعمل 3 Roles: Admin, Doctor, Patient
      ↓
SeedAdmin()              → admin@mediqueue.com / Admin@123
      ↓
SeedDoctorsAndClinics()  → 3 عيادات + 5 أطباء + مريض تجريبي
```

### حسابات تجريبية جاهزة

| الدور | الإيميل | الباسورد |
|---|---|---|
| Admin | admin@mediqueue.com | Admin@123 |
| Doctor | dr.khaled@mediqueue.com | Doctor@123 |
| Patient | patient@mediqueue.com | Patient@123 |

---

## ثالثاً: BL — طبقة منطق العمل

فولدر `BL` فيه **Interface + Implementation** لكل خدمة.

### الخدمات الموجودة

| Interface | الوظيفة |
|---|---|
| `IClinicService` | CRUD للعيادات |
| `IUserService` | جلب بيانات المستخدمين |
| `IAppointmentService` | حجز وإدارة المواعيد |
| `IQueueService` | إدارة الدور وإعادة الترتيب |

### IAppointmentService — أهم الوظائف

```csharp
Task<Appointment> CreateAppointmentAsync(Appointment appointment);
Task<bool> CancelAppointmentAsync(int appointmentId);
Task<bool> IsTimeSlotAvailableAsync(string doctorId, DateTime date, TimeSpan time);
Task<IEnumerable<Appointment>> GetAppointmentsByPatientAsync(string patientId);
Task<IEnumerable<Appointment>> GetAppointmentsByDoctorAsync(string doctorId);
```

### IQueueService — أهم الوظائف

```csharp
Task<int> GetNextPositionAsync(string doctorId);  // رقم الدور الجاي
Task ReorderQueueAsync(string doctorId);           // إعادة ترتيب الأرقام
Task<bool> DeactivateQueueAsync(int queueId);      // خروج من الدور
Task<IEnumerable<Queue>> GetActiveQueuesAsync(string doctorId);
```

### ليه Interface وليس Class مباشرة؟

الـ Controller ما بيعرفش التفاصيل، بيتعامل مع الـ Interface بس.
ده اسمه **Dependency Inversion** — لو غيرنا الـ Implementation (مثلاً من SQL لـ MongoDB)، الـ Controller ما هيتغيرش.

---

## رابعاً: Controllers — المحركات

### AccountController — تسجيل الدخول والخروج

#### تدفق تسجيل الدخول

```
المستخدم يكتب Email + Password
      ↓
POST /Account/Login
      ↓
_signInManager.PasswordSignInAsync()
      ↓
لو نجح → نشوف الـ Role
      ├── Admin   → /Admin/Index
      ├── Doctor  → /Doctor/Dashboard
      └── Patient → /Home/Index
لو فشل → "البريد أو الباسورد غير صحيح"
```

#### تسجيل الحساب (Register)

```csharp
// المستخدم الجديد دايماً بياخد Role = "Patient"
// الطبيب والأدمن بيتعملوا عن طريق DbInitializer
await _userManager.AddToRoleAsync(user, "Patient");
```

### AppointmentsController — حجز المواعيد

#### تدفق الحجز الكامل

```
[Patient] يفتح صفحة الطبيب
      ↓
GET /Appointments/Book?doctorId=xxx
      ↓
بيجيب بيانات الطبيب + بيانات المريض تلقائياً
      ↓
[Patient] يملأ التاريخ والسبب ويضغط حجز
      ↓
POST /Appointments/Book
      ↓
      ├── بيعمل Appointment جديد (Status = Confirmed)
      │
      └── بيعمل Queue جديد تلقائياً
          ├── Position = GetNextPositionAsync() (آخر رقم + 1)
          └── EstimatedTime = التاريخ + الوقت
      ↓
Redirect → /Appointments/Track/{id}
```

#### GetQueueStatus — نقطة الـ API للـ Polling

```csharp
// المريض في صفحة التتبع بيسأل كل X ثانية
GET /Appointments/GetQueueStatus?id=5
// بيرجع JSON:
{
    "position": 3,
    "peopleAhead": 2,
    "estimatedTime": "10:30 AM",
    "isActive": true
}
```

الفكرة: الـ Frontend بيعمل **Polling** (يسأل كل فترة) عشان يحدّث رقم الدور تلقائياً.

### DoctorController — لوحة الطبيب

```csharp
// Dashboard بيجيب مواعيد اليوم فقط
var todayAppointments = appointments
    .Where(a => a.AppointmentDate.Date == DateTime.Today)
    .OrderBy(a => a.AppointmentTime)
    .ToList();

ViewData["TotalToday"] = todayAppointments.Count;
ViewData["Upcoming"]   = upcomingCount;
ViewData["Completed"]  = completedCount;
```

#### CallNext — استدعاء المريض التالي

```
الطبيب يضغط "استدعاء التالي"
      ↓
POST /Doctor/CallNext
      ↓
بياخد أول حد في القائمة (Position = أصغر رقم)
      ↓
      ├── Appointment.Status = Completed
      └── ReorderQueueAsync() → بيعيد ترقيم الباقيين (2→1, 3→2...)
      ↓
Redirect → /Doctor/Queue
```

### QueuesController — إدارة الطابور

| Action | الوظيفة |
|---|---|
| `Index()` | كل الطوابير |
| `DoctorQueue(doctorId)` | طابور طبيب معين |
| `ActiveQueue(doctorId)` | الفعّالين فقط |
| `Edit(id)` | تعديل الموقع أو الوقت |
| `Deactivate(id)` | إخراج من الطابور |
| `Reorder(doctorId)` | إعادة ترتيب الأرقام |

---

## خامساً: ViewModel — نماذج البيانات

### ليه ViewModels وليس Models مباشرة؟

| Model (Entity) | ViewModel |
|---|---|
| كل الحقول موجودة | بس الحقول اللي الـ Form محتاجها |
| فيها FK و Navigation Properties | سهل للـ Validation |
| بتروح لقاعدة البيانات | بتيجي من الـ Form |

### AppointmentVM مثال

```csharp
public class AppointmentVM
{
    public string DoctorID { get; set; }
    public string DoctorName { get; set; }     // عرض بس، مش في DB
    public DateTime AppointmentDate { get; set; }
    public TimeSpan AppointmentTime { get; set; }
    public string? Reason { get; set; }
    public string? PatientName { get; set; }
    public string? PatientPhone { get; set; }
}
```

---

## سادساً: Program.cs — نقطة البداية

### ترتيب تسجيل الخدمات

```csharp
// 1. قاعدة البيانات
builder.Services.AddDbContext<MediQueueContext>(options =>
    options.UseSqlServer(connectionString));

// 2. نظام التأمين
builder.Services.AddIdentity<User, IdentityRole>(options => {
    options.Password.RequiredLength = 6;
})
.AddEntityFrameworkStores<MediQueueContext>();

// 3. إعدادات الـ Cookie
builder.Services.ConfigureApplicationCookie(options => {
    options.LoginPath = "/Account/Login";
    options.ExpireTimeSpan = TimeSpan.FromDays(7);
});

// 4. الـ Business Logic Services
builder.Services.AddScoped<IClinicService, ClinicService>();
builder.Services.AddScoped<IAppointmentService, AppointmentService>();
builder.Services.AddScoped<IQueueService, QueueService>();
```

### ليه `AddScoped`؟

```
AddSingleton  → instance واحد طول عمر التطبيق
AddScoped     → instance جديد لكل HTTP Request  ← ده المستخدم هنا
AddTransient  → instance جديد لكل مرة بتطلبه
```

### ترتيب الـ Middleware مهم جداً

```csharp
app.UseRouting();
app.UseAuthentication();   // لازم قبل Authorization
app.UseAuthorization();
```

---

## تدفق البيانات الكامل — سيناريو الحجز

```
[Browser] المريض يضغط "احجز"
      ↓
HTTP POST /Appointments/Book
      ↓
AppointmentsController.Book(AppointmentVM model)
      ↓
      ├── IAppointmentService.CreateAppointmentAsync()
      │         ↓
      │   AppointmentService → MediQueueContext
      │         ↓
      │   INSERT INTO Appointments ...
      │
      ├── IQueueService.GetNextPositionAsync(doctorId)
      │         ↓
      │   SELECT MAX(Position) FROM Queues WHERE DoctorID = x
      │
      └── IQueueService.CreateQueueAsync(queue)
                ↓
          INSERT INTO Queues (Position = max+1, IsActive = true)
      ↓
RedirectToAction("Track", id)
      ↓
[Browser] صفحة تتبع الدور
      ↓
JavaScript Polling كل 10 ثواني
      ↓
GET /Appointments/GetQueueStatus?id=X
      ↓
JSON { position, peopleAhead, estimatedTime }
```

---

## نظام الصلاحيات (Roles)

```
┌──────────┬──────────────────┬───────────────┐
│  Admin   │      Doctor      │    Patient    │
├──────────┼──────────────────┼───────────────┤
│ كل شيء  │ Dashboard        │ حجز موعد      │
│ إدارة   │ Queue بتاعه      │ تتبع الدور    │
│ المستخدم│ CallNext         │ إلغاء موعد    │
│ إدارة   │ مشاهدة مرضاه    │ مشاهدة        │
│ العيادات│ فقط              │ مواعيده فقط   │
└──────────┴──────────────────┴───────────────┘
```

### التطبيق في الكود

```csharp
// على مستوى الـ Controller كله
[Authorize(Roles = "Doctor")]
public class DoctorController : Controller { }

// في الـ Action نفسه
if (appointment.PatientID != user!.Id && !User.IsInRole("Admin"))
    return Forbid();  // المريض يشوف موعده بس
```

---

## أسئلة المناقشة المتوقعة

### أسئلة أساسية

**س: إيه هو MVC وكيف بيشتغل في مشروعك؟**

MVC = Model, View, Controller.
- **Model**: البيانات (Appointment, User, Clinic)
- **View**: الصفحة (cshtml)
- **Controller**: بيستقبل الطلب، بيكلم الـ Model، وبيرجع الـ View

```
Browser → Controller → Model (DB) → Controller → View → Browser
```

---

**س: إيه Entity Framework وليه استخدمتوه؟**

ORM بيحوّل C# لـ SQL تلقائياً:
```csharp
// بدل ما نكتب SQL يدوي
await _context.Appointments
    .Where(a => a.PatientID == userId)
    .ToListAsync();
```

---

**س: إيه الـ Dependency Injection؟**

```csharp
// ❌ بدون DI
var service = new AppointmentService(new MediQueueContext(...));

// ✅ مع DI — الـ Framework بيحقنه تلقائياً
public AppointmentsController(IAppointmentService appointmentService)
{
    _appointmentService = appointmentService;
}
```

الفايدة: لو غيرنا الـ Implementation، الـ Controller ما بيتغيرش.

---

**س: إيه الفرق بين `[Authorize]` و `[Authorize(Roles = "Doctor")]`؟**

```csharp
[Authorize]                        // أي مستخدم مسجّل دخول
[Authorize(Roles = "Doctor")]      // طبيب فقط
[Authorize(Roles = "Admin,Doctor")]// أدمن أو طبيب
// بدون Attribute = الكل يدخل
```

---

### أسئلة متوسطة

**س: إزاي بيشتغل الـ Queue tracking؟**

```
صفحة Track بتفتح
      ↓
JavaScript setInterval كل 10 ثواني
      ↓
fetch('/Appointments/GetQueueStatus?id=5')
      ↓
Controller بيرجع JSON { position, peopleAhead }
      ↓
JavaScript بيحدّث الرقم من غير page refresh
```

ده اسمه **Polling** — أبسط من WebSockets لكن فيه overhead أكتر.

---

**س: ليه استخدمنا `OnDelete(DeleteBehavior.Restrict)`؟**

لو استخدمنا Cascade ومسحنا طبيب، كل مواعيده هتتمسح.
الـ Restrict بيمنع المسح لو فيه بيانات مرتبطة — ده أأمن للبيانات الطبية.

---

**س: ليه الـ AppointmentStatus محفوظ كـ String في DB؟**

```csharp
.Property(a => a.Status).HasConversion<string>();
```

بدل ما يتحفظ كرقم (0,1,2,3)، بيتحفظ "Booked", "Confirmed"...
لو بصنا على الـ DB مباشرة هنفهم البيانات من غير ما نرجع للكود.

---

### أسئلة متقدمة

**س: إيه حدود النظام (Limitations)؟**

| المشكلة | السبب | الحل المثالي |
|---|---|---|
| Polling كل 10 ثواني | مش Real-time | SignalR / WebSockets |
| مفيش تنبيهات | مفيش Push Notifications | Firebase / Email |
| AvailableSlots كـ String | مش مرنة | جدول منفصل للـ Slots |
| مفيش Pagination | هيبطّأ مع كتير بيانات | Server-side Pagination |

---

**س: إيه الـ ReorderQueueAsync وليه مهمة؟**

لما الطبيب يستدعي التالي، المريض الأول بيخرج:
```
قبل:  [1:أحمد, 2:محمد, 3:سارة]
بعد مسح أحمد بدون reorder: [2:محمد, 3:سارة]  ← مش صح
بعد ReorderQueueAsync:      [1:محمد, 2:سارة]  ← صح
```

---

**س: ليه `AddScoped` للـ Services؟**

الـ DbContext مصمم لـ Request واحد.
الـ Services بتستخدم DbContext، فلازم نفس العمر.
لو استخدمنا Singleton → هيتشارك DbContext بين Requests → مشاكل Threading.

---

## الملخص الكامل

| الملف | الدور |
|---|---|
| `User.cs` | جدول واحد للمريض والطبيب والأدمن |
| `Appointment.cs` | الحجز الرابط بين مريض وطبيب |
| `Queue.cs` | رقم الدور وحالته |
| `MediQueueContext.cs` | حلقة الوصل مع SQL Server |
| `DbInitializer.cs` | بيانات تجريبية عند أول تشغيل |
| `IAppointmentService` | عقد وظائف المواعيد |
| `IQueueService` | عقد وظائف الطابور |
| `AccountController` | تسجيل الدخول والخروج والتسجيل |
| `AppointmentsController` | حجز وتتبع وإلغاء المواعيد |
| `DoctorController` | Dashboard الطبيب + استدعاء التالي |
| `QueuesController` | إدارة الطابور CRUD |
| `ClinicsController` | إدارة العيادات CRUD |
| `Program.cs` | تسجيل الخدمات ونقطة البداية |