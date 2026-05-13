# Hall Dining Token Management System - Updated Requirements

**Project:** Hall Dining Token Management System  
**Version:** Software Project 2 MVP  
**Document Type:** Requirements Specification  
**Updated Requirement:** BatchAdmin replaced with CashAdmin / Authorized Cash Admin  
**Technology:** ASP.NET Core MVC, Razor Views, Entity Framework Core, ASP.NET Core Identity, SQL Server  
**Prepared For:** Visual Studio + GitHub Copilot implementation workflow  
**Date:** 06 May 2026  

---

## 1. Project Overview

The Hall Dining Token Management System is a role-based web application designed to digitize the manual hall dining meal token process.

The system will allow students to register, maintain a wallet, purchase meal tokens using wallet balance, receive OTP-based tokens, and redeem meals through OTP verification by a delivery user.

For Software Project 2, the system will remain simple, practical, and demo-friendly. It will use an ASP.NET Core MVC single-project structure with Razor Views, Entity Framework Core, ASP.NET Core Identity, and SQL Server.

---

## 2. Main MVP Flow

```text
Landing Page
-> Register / Login
-> SuperAdmin Creates Meal
-> CashAdmin Searches Student by Student ID
-> CashAdmin Tops Up Student Wallet
-> Cash Statement Updates
-> Student Buys Meal Token
-> OTP Token Generated
-> DeliveryMan Verifies OTP
-> Token Redeemed
-> Admin Report Updated
```

---

## 3. Updated Requirement Summary

The previous role name **BatchAdmin** is removed.

The new role will be:

```text
Internal Role Name: CashAdmin
Display Name: Authorized Cash Admin
```

### Reason for Change

The old BatchAdmin model was too restrictive because it allowed wallet top-up only for students of the same department and batch.

The updated system is more flexible:

```text
Any Authorized Cash Admin can top-up any registered student's wallet.
The system records which CashAdmin collected the cash.
```

This keeps the system practical while maintaining accountability through `WalletTransaction` and `CashCollection` records.

---

## 4. User Roles

| Role | Display Name | Main Responsibility |
|---|---|---|
| SuperAdmin | Super Admin | Meal creation, cash settlement, reports, system overview |
| CashAdmin | Authorized Cash Admin | Student wallet top-up, cash collection tracking, cash statement |
| Student | Student | Register, view wallet, buy tokens, view OTP tokens |
| DeliveryMan | Delivery Man | Verify OTP and redeem meal token |

---

## 5. Functional Requirements

## 5.1 Public Landing Page

The system shall provide a public landing page.

### Requirements

- User can open the root URL `/` without login.
- Landing page shall show:
  - Project title
  - Short problem statement
  - Main features
  - Role overview
  - Login button
  - Register button
  - Future enhancement note

---

## 5.2 Authentication and Registration

The system shall use ASP.NET Core Identity for authentication.

### Requirements

- User can register as a Student.
- Newly registered users shall receive the `Student` role by default.
- Student registration form shall collect:
  - Full Name
  - Email
  - Student ID
  - Department
  - Batch
  - Password
  - Confirm Password
- Student ID must be unique.
- A wallet with balance `0` shall be created automatically after successful student registration.
- Users cannot register themselves as SuperAdmin, CashAdmin, or DeliveryMan.

---

## 5.3 Role-Based Access

The system shall restrict pages based on user role.

### Requirements

- SuperAdmin can access only SuperAdmin pages.
- CashAdmin can access only wallet top-up and cash statement pages.
- Student can access only own wallet, meals, orders, and tokens.
- DeliveryMan can access only OTP verification and redeem pages.
- Unauthorized users shall be blocked from restricted routes.

---

## 5.4 Meal Management

SuperAdmin shall be able to create and manage meal sessions.

### Meal Fields

| Field | Description |
|---|---|
| MealName | Name of the meal |
| MealType | Breakfast, Lunch, Dinner |
| MealDate | Date of the meal |
| Price | Price per token |
| Capacity | Maximum number of tokens available |
| SoldCount | Number of tokens already sold |
| SaleEndTime | Deadline for token purchase |

### Requirements

- SuperAdmin can create a meal.
- SuperAdmin can view meal list.
- Student can see only available meals.
- A meal is available only if:
  - SaleEndTime has not passed.
  - SoldCount is less than Capacity.

---

## 5.5 Wallet System

Each student shall have one wallet.

### Requirements

- Wallet balance shall increase after CashAdmin top-up.
- Wallet balance shall decrease after successful token purchase.
- Every balance change shall create a `WalletTransaction`.
- Student can view wallet balance.
- Student can view wallet transaction history.

---

## 5.6 CashAdmin Wallet Top-up

CashAdmin shall top-up student wallet by searching the student using unique Student ID.

### Updated Top-up Flow

```text
CashAdmin opens Wallet Top-up page
-> Enters Student ID
-> Clicks Search
-> System finds student
-> System shows student details and current wallet balance
-> CashAdmin enters amount and optional note
-> CashAdmin submits top-up
-> Wallet balance increases
-> WalletTransaction is created
-> CashCollection is created under current CashAdmin
-> Cash statement is updated
```

### Requirements

- CashAdmin must search student by Student ID.
- System shall show student details after successful search:
  - Full Name
  - Student ID
  - Department
  - Batch
  - Current Wallet Balance
- CashAdmin can top-up any registered student.
- There shall be no department or batch restriction.
- CashAdmin must enter a positive top-up amount.
- Every top-up must create:
  - `WalletTransaction`
  - `CashCollection`
- `CashCollection` must store the `CashAdminId`.
- Cash statement shall be calculated based on the current CashAdmin's collections and settlements.

---

## 5.7 Cash Statement

CashAdmin shall be able to see personal cash statement.

### Cash Statement Metrics

| Metric | Calculation |
|---|---|
| Total Collected | Sum of CashCollections where CashAdminId = current user |
| Total Settled | Sum of CashSettlements where CashAdminId = current user |
| Pending Cash | Total Collected - Total Settled |
| Total Top-ups | Count of CashCollections by current CashAdmin |

### Requirements

- CashAdmin can view total collected cash.
- CashAdmin can view total settled cash.
- CashAdmin can view pending cash.
- CashAdmin can view top-up history.
- CashAdmin can view settlement history.

---

## 5.8 Cash Settlement

SuperAdmin shall record cash received from CashAdmin.

### Requirements

- SuperAdmin can select a CashAdmin.
- SuperAdmin can enter settlement amount.
- Settlement amount must be greater than 0.
- Settlement amount must not exceed that CashAdmin's pending cash.
- Settlement record shall create a `CashSettlement`.
- CashAdmin pending cash shall decrease after settlement.

---

## 5.9 Token Purchase

Student shall buy meal tokens using wallet balance.

### Requirements

- Student can buy token from available meals.
- Student can buy minimum 1 token at a time.
- Student can buy maximum 4 tokens at a time.
- System must validate:
  - Meal exists.
  - Quantity is between 1 and 4.
  - SaleEndTime has not passed.
  - Capacity is available.
  - Wallet balance is sufficient.
- Successful purchase shall:
  - Deduct wallet balance.
  - Create an `Order`.
  - Create one `MealToken` per quantity.
  - Generate unique 6-digit OTP for each token.
  - Increase `MealSession.SoldCount`.
  - Create a negative `WalletTransaction`.

---

## 5.10 OTP Token System

The system shall generate OTP tokens after successful purchase.

### Requirements

- OTP shall be 6 digits.
- OTP shall be unique.
- Student can see own OTP tokens.
- Token status shall be:
  - Active
  - Redeemed
- OTP can be used only once.
- Redeemed token cannot be redeemed again.

---

## 5.11 Delivery Verification

DeliveryMan shall verify and redeem meal tokens by OTP.

### Flow

```text
DeliveryMan opens Verify page
-> Enters OTP
-> System searches token
-> If OTP is invalid, show error
-> If token is redeemed, show already redeemed message
-> If token is active, show student and meal details
-> DeliveryMan clicks Mark as Delivered
-> Token status changes to Redeemed
```

### Requirements

- DeliveryMan can enter OTP.
- System shall show token details for valid active OTP.
- System shall block invalid OTP.
- System shall block already redeemed OTP.
- Redeem action shall update:
  - Token status
  - RedeemedByUserId
  - RedeemedAt

---

## 5.12 Admin Reports

SuperAdmin shall view basic system reports.

### Report Metrics

| Metric | Source |
|---|---|
| Total Meals | MealSessions count |
| Total Tokens Sold | MealTokens count |
| Total Revenue | Orders TotalAmount sum |
| Active Tokens | MealTokens where Status = Active |
| Redeemed Tokens | MealTokens where Status = Redeemed |
| Total Wallet Top-up | WalletTransactions where Type = TopUp |
| Total Cash Collected | CashCollections amount sum |
| Total Cash Settled | CashSettlements amount sum |
| Pending Cash | Total Cash Collected - Total Cash Settled |

---

## 6. Business Rules

| Rule | Description |
|---|---|
| Default Role | New registered user receives Student role |
| Unique Student ID | Student ID must be unique |
| Wallet Auto Creation | Student wallet is created during registration |
| CashAdmin Flexibility | CashAdmin can top-up any registered student |
| Student Search | CashAdmin must search student by Student ID |
| Top-up Tracking | Every top-up creates WalletTransaction and CashCollection |
| Max Token Purchase | Student can buy maximum 4 tokens at a time |
| Min Token Purchase | Student must buy at least 1 token |
| Sale Deadline | Token cannot be purchased after SaleEndTime |
| Capacity Limit | SoldCount + Quantity must not exceed Capacity |
| Balance Check | Wallet balance must be sufficient |
| Unique OTP | OTP must be unique |
| One-time Redemption | Redeemed OTP cannot be used again |
| Settlement Limit | Settlement amount cannot exceed pending cash |

---

## 7. Database Entities

## 7.1 ApplicationUser

```csharp
public class ApplicationUser : IdentityUser
{
    public string FullName { get; set; } = string.Empty;
    public string? StudentId { get; set; }
    public string? Department { get; set; }
    public string? Batch { get; set; }
}
```

## 7.2 Wallet

```csharp
public class Wallet
{
    public int Id { get; set; }
    public string UserId { get; set; } = string.Empty;
    public ApplicationUser? User { get; set; }
    public decimal Balance { get; set; }
}
```

## 7.3 WalletTransaction

```csharp
public class WalletTransaction
{
    public int Id { get; set; }
    public int WalletId { get; set; }
    public Wallet? Wallet { get; set; }
    public decimal Amount { get; set; }
    public string Type { get; set; } = string.Empty; // TopUp, MealPurchase
    public string? CreatedByUserId { get; set; }
    public string? Note { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.Now;
}
```

## 7.4 MealSession

```csharp
public class MealSession
{
    public int Id { get; set; }
    public string MealName { get; set; } = string.Empty;
    public string MealType { get; set; } = string.Empty;
    public DateTime MealDate { get; set; }
    public decimal Price { get; set; }
    public int Capacity { get; set; }
    public int SoldCount { get; set; }
    public DateTime SaleEndTime { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.Now;
}
```

## 7.5 Order

```csharp
public class Order
{
    public int Id { get; set; }
    public string UserId { get; set; } = string.Empty;
    public ApplicationUser? User { get; set; }
    public int MealSessionId { get; set; }
    public MealSession? MealSession { get; set; }
    public int Quantity { get; set; }
    public decimal TotalAmount { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.Now;
}
```

## 7.6 MealToken

```csharp
public class MealToken
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public Order? Order { get; set; }
    public int MealSessionId { get; set; }
    public MealSession? MealSession { get; set; }
    public string UserId { get; set; } = string.Empty;
    public ApplicationUser? User { get; set; }
    public string OtpCode { get; set; } = string.Empty;
    public string Status { get; set; } = "Active";
    public string? RedeemedByUserId { get; set; }
    public DateTime? RedeemedAt { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.Now;
}
```

## 7.7 CashCollection

```csharp
public class CashCollection
{
    public int Id { get; set; }

    public string CashAdminId { get; set; } = string.Empty;
    public ApplicationUser? CashAdmin { get; set; }

    public string StudentId { get; set; } = string.Empty;
    public ApplicationUser? Student { get; set; }

    public decimal Amount { get; set; }

    public int WalletTransactionId { get; set; }
    public WalletTransaction? WalletTransaction { get; set; }

    public DateTime CreatedAt { get; set; } = DateTime.Now;
}
```

## 7.8 CashSettlement

```csharp
public class CashSettlement
{
    public int Id { get; set; }

    public string CashAdminId { get; set; } = string.Empty;
    public ApplicationUser? CashAdmin { get; set; }

    public decimal Amount { get; set; }

    public string ReceivedByAdminId { get; set; } = string.Empty;
    public ApplicationUser? ReceivedByAdmin { get; set; }

    public string? Note { get; set; }
    public DateTime SettlementDate { get; set; } = DateTime.Now;
}
```

---

## 8. DbContext Requirements

`ApplicationDbContext` shall inherit from:

```csharp
IdentityDbContext<ApplicationUser>
```

### Required DbSets

```csharp
public DbSet<MealSession> MealSessions => Set<MealSession>();
public DbSet<Wallet> Wallets => Set<Wallet>();
public DbSet<WalletTransaction> WalletTransactions => Set<WalletTransaction>();
public DbSet<Order> Orders => Set<Order>();
public DbSet<MealToken> MealTokens => Set<MealToken>();
public DbSet<CashCollection> CashCollections => Set<CashCollection>();
public DbSet<CashSettlement> CashSettlements => Set<CashSettlement>();
```

### Required Indexes

```csharp
builder.Entity<ApplicationUser>()
    .HasIndex(u => u.StudentId)
    .IsUnique()
    .HasFilter("[StudentId] IS NOT NULL");

builder.Entity<Wallet>()
    .HasIndex(w => w.UserId)
    .IsUnique();

builder.Entity<MealToken>()
    .HasIndex(t => t.OtpCode)
    .IsUnique();
```

---

## 9. Controllers and Routes

## 9.1 Public

| Route | Controller | Purpose |
|---|---|---|
| `/` | HomeController | Landing page |
| `/Identity/Account/Register` | Identity | Student registration |
| `/Identity/Account/Login` | Identity | Login |

## 9.2 SuperAdmin

| Route | Purpose |
|---|---|
| `/Admin/Dashboard` | Admin dashboard |
| `/Admin/Meals` | Meal list |
| `/Admin/CreateMeal` | Create meal |
| `/Admin/Reports` | Reports |
| `/Admin/CreateSettlement` | Record cash settlement |

## 9.3 CashAdmin

| Route | Purpose |
|---|---|
| `/CashAdmin/Dashboard` | CashAdmin dashboard |
| `/CashAdmin/TopUpWallet` | Search student and top-up wallet |
| `/CashAdmin/CashStatement` | Cash statement |

## 9.4 Student

| Route | Purpose |
|---|---|
| `/Student/Dashboard` | Student dashboard |
| `/Student/Meals` | Available meals |
| `/Student/BuyToken` | Token purchase POST |
| `/Student/MyTokens` | Purchased OTP tokens |
| `/Student/Wallet` | Wallet balance and transactions |

## 9.5 DeliveryMan

| Route | Purpose |
|---|---|
| `/Delivery/Dashboard` | Delivery dashboard |
| `/Delivery/Verify` | OTP verification |
| `/Delivery/Redeem` | Mark token as delivered |

---

## 10. Service Layer Requirements

## 10.1 MealService

Responsibilities:

- Create meal.
- Get all meals.
- Get available meals.
- Validate future SaleEndTime.

## 10.2 WalletService

Responsibilities:

- Top-up student wallet.
- Validate positive amount.
- Create wallet if missing.
- Create WalletTransaction.
- Create CashCollection.
- Use database transaction.

## 10.3 TokenService

Responsibilities:

- Buy token.
- Validate quantity 1 to 4.
- Validate sale deadline.
- Validate capacity.
- Validate wallet balance.
- Deduct wallet balance.
- Create order.
- Create OTP meal tokens.
- Redeem token.
- Prevent duplicate redemption.

## 10.4 ReportService

Responsibilities:

- Calculate SuperAdmin report metrics.
- Calculate CashAdmin cash statement.
- Calculate pending cash.
- Return dashboard summaries.

---

## 11. Required ViewModels

| ViewModel | Purpose |
|---|---|
| CreateMealViewModel | Meal create form |
| WalletTopUpViewModel | Student ID search and wallet top-up |
| BuyTokenViewModel | Token purchase |
| VerifyOtpViewModel | OTP verification |
| CashStatementViewModel | CashAdmin cash statement |
| DashboardViewModel | Dashboard cards |
| SettlementViewModel | SuperAdmin settlement form |

---

## 12. WalletTopUpViewModel

```csharp
public class WalletTopUpViewModel
{
    [Required]
    [Display(Name = "Student ID")]
    public string StudentId { get; set; } = string.Empty;

    public string? StudentUserId { get; set; }

    public string? StudentName { get; set; }
    public string? Department { get; set; }
    public string? Batch { get; set; }
    public decimal? CurrentBalance { get; set; }

    [Range(1, 100000)]
    public decimal Amount { get; set; }

    public string? Note { get; set; }
}
```

---

## 13. UI Requirements

## 13.1 General UI

- Use Bootstrap.
- Use cards for dashboard summaries.
- Use tables for records.
- Use forms for input.
- Use badges for token status.
- Use TempData alerts for success/error messages.

## 13.2 CashAdmin Top-up UI

The top-up page shall include:

1. Student ID input field.
2. Search Student button.
3. Student details card after search.
4. Amount input.
5. Optional note input.
6. Top-up Wallet button.

---

## 14. Validation Requirements

| Validation | Location |
|---|---|
| Student ID required | Register, Top-up search |
| Student ID unique | Register, Database index |
| Amount positive | WalletTopUpViewModel, WalletService |
| Quantity 1-4 | BuyTokenViewModel, TokenService |
| Sufficient balance | TokenService |
| Capacity available | TokenService |
| SaleEndTime valid | MealService, TokenService |
| OTP unique | TokenService, Database index |
| Settlement not greater than pending cash | ReportService/AdminController |
| Duplicate redemption blocked | TokenService |

---

## 15. Data Integrity Requirements

The following operations must use database transactions:

## 15.1 Wallet Top-up

```text
Begin Transaction
-> Update Wallet Balance
-> Create WalletTransaction
-> Create CashCollection
-> Commit
```

If any step fails, rollback.

## 15.2 Token Purchase

```text
Begin Transaction
-> Validate Meal
-> Validate Wallet
-> Deduct Wallet Balance
-> Create Order
-> Create MealTokens
-> Update SoldCount
-> Create WalletTransaction
-> Commit
```

If any step fails, rollback.

---

## 16. Seed Data Requirements

The system shall seed the following roles:

```text
SuperAdmin
CashAdmin
Student
DeliveryMan
```

### Demo Accounts

| Role | Email | Password |
|---|---|---|
| SuperAdmin | superadmin@test.com | Test@123 |
| CashAdmin | cashadmin@test.com | Test@123 |
| Student | student1@test.com | Test@123 |
| Student | student2@test.com | Test@123 |
| DeliveryMan | delivery@test.com | Test@123 |

Students shall have wallets created during seed.

---

## 17. Implementation Order

| Phase | Task |
|---|---|
| Phase 0 | Create ASP.NET Core MVC project with Individual Accounts |
| Phase 1 | Customize ApplicationUser and Register page |
| Phase 2 | Add database entities |
| Phase 3 | Configure ApplicationDbContext and migrations |
| Phase 4 | Seed roles and demo users |
| Phase 5 | Create landing page |
| Phase 6 | Implement SuperAdmin meal create/list |
| Phase 7 | Implement CashAdmin Student ID search and wallet top-up |
| Phase 8 | Implement CashAdmin cash statement |
| Phase 9 | Implement SuperAdmin cash settlement |
| Phase 10 | Implement Student wallet and available meals |
| Phase 11 | Implement token purchase and OTP generation |
| Phase 12 | Implement DeliveryMan OTP verification and redeem |
| Phase 13 | Implement reports and dashboards |
| Phase 14 | Final testing and UI polish |

---

## 18. Testing Checklist

## 18.1 Authentication Tests

- New student can register.
- Student ID duplicate is blocked.
- New student receives Student role.
- New student wallet is created with balance 0.
- Demo users can login.

## 18.2 Role Tests

- Student cannot access Admin route.
- CashAdmin cannot access SuperAdmin route.
- DeliveryMan cannot access wallet top-up route.
- SuperAdmin can access reports.

## 18.3 CashAdmin Top-up Tests

- CashAdmin can search student by valid Student ID.
- Invalid Student ID shows error.
- Student details show after search.
- Top-up amount 500 increases wallet balance.
- WalletTransaction is created.
- CashCollection is created.
- CashAdmin cash statement updates.
- CashAdmin can top-up students from any department or batch.

## 18.4 Settlement Tests

- SuperAdmin can record settlement.
- Settlement amount greater than pending cash is blocked.
- Pending cash decreases after settlement.

## 18.5 Token Purchase Tests

- Student can buy 1 token.
- Student can buy 4 tokens.
- Student cannot buy 5 tokens.
- Insufficient balance blocks purchase.
- SaleEndTime passed blocks purchase.
- Capacity exceeded blocks purchase.
- OTP tokens are generated correctly.

## 18.6 Delivery Tests

- Valid OTP shows token details.
- Invalid OTP shows error.
- Redeem changes status to Redeemed.
- Same OTP again shows already redeemed.

## 18.7 Report Tests

- Tokens sold count is correct.
- Revenue is correct.
- Active token count is correct.
- Redeemed token count is correct.
- Collected, settled, and pending cash are correct.

---

## 19. Demo Script

```text
1. Open landing page.
2. Show problem, features, and Login/Register buttons.
3. Register a new student or login with seeded student.
4. Login as SuperAdmin.
5. Create a lunch meal: Rice + Chicken, price 70, capacity 100.
6. Login as CashAdmin.
7. Search student by Student ID.
8. Top-up wallet with BDT 500.
9. Show CashAdmin cash statement.
10. Login as Student.
11. Show wallet balance.
12. Buy 4 lunch tokens.
13. Try to buy 5 tokens and show validation error.
14. Show My Tokens page with OTP codes.
15. Login as DeliveryMan.
16. Verify one OTP.
17. Mark token as delivered.
18. Verify same OTP again and show already redeemed message.
19. Login as SuperAdmin.
20. Record cash settlement.
21. Show reports and updated pending cash.
```

---

## 20. Out of Scope for Software Project 2

The following features are not included in SP2:

- QR code generation.
- QR scanner.
- Online payment gateway.
- bKash/Nagad integration.
- Refund system.
- SMS/email notification.
- Food rating.
- Advanced analytics.
- Mobile app.
- PWA.
- Full accounting-grade reconciliation.

---

## 21. Future Enhancements for Software Project 3

- QR code token verification.
- Camera-based QR scanner.
- Online wallet top-up.
- Payment gateway integration.
- Student top-up confirmation.
- Refund and cancellation workflow.
- Token expiry by pickup time.
- OTP hashing.
- Audit logs.
- Food demand prediction.
- Advanced report dashboard.
- Mobile-friendly PWA.

---

## 22. Final Acceptance Criteria

The project will be considered complete for Software Project 2 if:

- Landing page works.
- Student registration works.
- Student ID is unique.
- Student wallet is auto-created.
- Role-based login works.
- SuperAdmin can create meals.
- CashAdmin can search student by Student ID.
- CashAdmin can top-up any student wallet.
- WalletTransaction and CashCollection are created.
- Cash statement works.
- SuperAdmin settlement works.
- Student can buy tokens using wallet balance.
- Maximum 4 token rule works.
- OTP tokens are generated.
- DeliveryMan can verify and redeem OTP.
- Duplicate redemption is blocked.
- Admin report shows correct values.
- Full demo flow runs without manual database editing.

---

## 23. Copilot Prompt Library

### Project Context Prompt

```text
I am building an ASP.NET Core MVC app called HallDiningTokenSystem.
It uses Identity, EF Core, Razor Views and SQL Server LocalDB.
The app has four roles: SuperAdmin, CashAdmin, Student, DeliveryMan.
CashAdmin is displayed as Authorized Cash Admin.
Please help me implement it step by step with simple service-layer architecture.
```

### CashAdmin Top-up Prompt

```text
Create a CashAdmin wallet top-up flow for my ASP.NET Core MVC app.
CashAdmin should search a student using unique StudentId.
After search, show student name, department, batch and current wallet balance.
Then CashAdmin enters amount and note.
On submit, call WalletService.TopUpAsync using the student's internal user id.
Do not restrict by department or batch.
Every top-up must create WalletTransaction and CashCollection.
```

### TokenService Prompt

```text
Create TokenService for buying and redeeming meal tokens.
BuyTokenAsync should validate quantity 1 to 4, sale deadline, meal capacity,
and wallet balance. Use a database transaction to deduct wallet balance,
create Order, create one MealToken with unique 6-digit OTP for each quantity,
update MealSession.SoldCount and create WalletTransaction.
Also implement VerifyOtpAsync and RedeemAsync to prevent duplicate redemption.
```

### Debugging Prompt

```text
Here is the error message and the relevant code.
Explain the likely cause in simple terms and give the smallest safe fix.
Do not rewrite unrelated files.
```

---

## 24. Notes

This requirements document reflects the updated Software Project 2 design.

The most important updated decisions are:

```text
BatchAdmin removed.
CashAdmin added.
CashAdmin means Authorized Cash Admin.
CashAdmin can top-up any registered student wallet.
Student search during top-up will be done by unique Student ID.
Department and Batch remain as student information fields, not hard authorization restrictions.
```
