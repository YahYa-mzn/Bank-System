# 🏦 Bank System

### Console Banking Application with Users, Permissions, Transactions & Currency Exchange

**A full-featured console application for managing bank clients, users, transactions and currency exchange — built with Object-Oriented C++ and plain text files as the database**

![C++](https://img.shields.io/badge/C%2B%2B-00599C?style=for-the-badge&logo=cplusplus&logoColor=white) ![OOP](https://img.shields.io/badge/OOP-8E44AD?style=for-the-badge) ![STL](https://img.shields.io/badge/STL-E67E22?style=for-the-badge) ![File I/O](https://img.shields.io/badge/File%20I%2FO-27AE60?style=for-the-badge) ![Console](https://img.shields.io/badge/Console-2C3E50?style=for-the-badge&logo=windowsterminal&logoColor=white)

**Version 1.0** — by [Yahya Mazini](https://github.com/YahYa-mzn)

---

## About This Project

This is my **first Object-Oriented Programming project**. It is a complete implementation of a **Bank System** based on **Course 11 by Mohammed Abu Hadhoud ([Programming Advices](https://www.youtube.com/@ProgrammingAdvices))**.

The goal was to take everything learned in the course and put it into one real application: designing classes, thinking about the business rules of a bank, and using **text files as a database**, all in a clean OOP structure.

It covers: client management, user management with a bitmask permission system, deposits / withdrawals / transfers, a transfer log, a login register, and a currency exchange module with a calculator.

**Concepts practiced in this project:**

| Concept | Where it shows up |
| --- | --- |
| **Encapsulation** | Private fields (`_AccountNumber`, `_Mode`, ...) exposed only through getters/setters and `__declspec(property)` properties |
| **Abstraction** | Screens never touch files — they call `Find()`, `Save()`, `Deposit()`, `Transfer()` and the class handles everything |
| **Inheritance** | `clsPerson` → `clsBankClient`, `clsUser`; every screen inherits from `clsScreen` |
| **Polymorphism / Interfaces** | `InterfaceCommunication` (pure virtual `SendEmail / SendSMS / SendFax`) implemented by `clsPerson` |
| **Static classes & utilities** | `clsString`, `clsDate`, `clsUtil`, `clsInputValidate<T>` |
| **Templates** | `clsInputValidate<T>` (`short`, `double`, ...) |
| **Value semantics & RAII** | Objects are passed by value / reference, files are opened and closed per operation — no manual `new` / `delete` |
| **Files as a database** | Line ↔ object conversion, load-to-vector, rewrite-file, append-line patterns |
| **Business thinking** | Permissions, admin protection, unique keys, balance checks, audit logs (login + transfers) |

### Development Timeline

**Version 1.0 — ~10–15 days**, working roughly 4–8 hours per day.

Built while following the course, practicing every concept as it was taught and applying it directly to the project instead of only watching.

---

## Architecture

The application is a **header-only** C++ project. The code is organized in three logical layers plus a shared utilities group. Each domain class follows the *Active Record* idea: **it knows how to find, save and delete itself** in its own text file.

```
┌─────────────────────────────────────────────────────┐
│            Presentation Layer (Screens)              │
│  clsLoginScreen, clsMainScreen, clsTransactionsScreen│
│  clsManageUsersScreen, clsCurrencyExchangeScreen ... │
│  All inherit from clsScreen (header + permissions).  │
└──────────────────────┬──────────────────────────────┘
                       │  calls only
┌──────────────────────▼──────────────────────────────┐
│               Domain Layer (Classes)                 │
│  clsPerson → clsBankClient / clsUser                 │
│  clsCurrency                                         │
│  Business rules, modes, Save() results, permissions  │
└──────────────────────┬──────────────────────────────┘
                       │  reads / writes
┌──────────────────────▼──────────────────────────────┐
│              Storage (Text Files)                    │
│  Clients.txt  Users.txt  Currencies.txt              │
│  LoginRegister.txt  TransferLog.txt                  │
└─────────────────────────────────────────────────────┘

        Shared utilities: clsInputValidate<T>, clsString,
                          clsDate, clsUtil, Global.h
```

### Class Hierarchy

```
InterfaceCommunication  (abstract: SendEmail / SendSMS / SendFax)
        │
    clsPerson           (FirstName, LastName, Email, Phone, FullName())
        ├── clsBankClient   (AccountNumber, PinCode, AccountBalance, Transfers)
        └── clsUser         (UserName, Password, Permissions, Login Register)

clsScreen               (header, current user/date, _HasPermission)
        ├── clsLoginScreen
        ├── clsMainScreen
        ├── clsTransactionsScreen / clsCurrencyExchangeScreen / clsManageUsersScreen
        └── one screen per operation (Deposit, Withdraw, Transfer, Find, Update ...)
```

### Navigation Flow

```
Login ──(3 attempts)──► Main Menu
                          ├── [1-5] Clients  (List / Add / Delete / Update / Find)
                          ├── [6]   Transactions
                          │          ├── Deposit
                          │          ├── Withdraw
                          │          ├── Total Balances
                          │          ├── Transfer
                          │          └── Transfer Log
                          ├── [7]   Currency Exchange
                          │          ├── List Currencies
                          │          ├── Find Currency
                          │          ├── Update Rate
                          │          └── Currency Calculator
                          ├── [8]   Manage Users (List / Add / Delete / Update / Find)
                          ├── [9]   Login Register
                          ├── [10]  Logout
                          └── [11]  Exit
```

---

## Files as a Database

Instead of a real database, every entity is stored in a **plain text file**, one record per line, fields separated by `#//#`. Each persistent class uses the same small set of private helpers:

| Helper | Purpose |
| --- | --- |
| `_ConvertLineToObject` | Split a line by the separator and build an object (`UpdateMode`) |
| `_ConvertObjectToLine` | Serialize an object back to a line |
| `_LoadDataFromFileToVector` | Read the whole file into a `vector` of objects |
| `_SaveDataInFile` | Rewrite the file from a vector (skips objects flagged `MarkedForDelete`) |
| `_AddDataLineToFile` | Append a single new line (`ios::app`) |
| `_Update` / `_AddNew` | Replace one record / append one record |

```cpp
// Updating a client: load all, replace the matching one, write all back
void _Update()
{
    vector<clsBankClient> _vClients = _LoadDataFromFileToVector();

    for (clsBankClient& _Client : _vClients)
    {
        if (_Client.AccountNumber() == AccountNumber())
        {
            _Client = *this;
            break;
        }
    }

    _SaveClientsDataInFile(_vClients);
}
```

### Object Modes

Every object carries a private **mode** that decides what `Save()` does, so the screens never have to care whether they are adding or updating:

| Mode | Meaning | `Save()` behavior |
| --- | --- | --- |
| `EmptyMode` | Nothing was found (the "null object") | Fails with `svFailedEmptyObject` |
| `UpdateMode` | Loaded from the file | Rewrites the file with the updated record |
| `AddNewMode` | Created by `GetAddNewClientObject()` / `GetAddNewUserObject()` | Checks uniqueness, appends the line, then switches to `UpdateMode` |

`Save()` returns an `enSaveResults` value (`svSucceeded`, `svFailedEmptyObject`, `svFailedAccNumExists`, `svFailedUserNameExists`, `svFailedAdmin`) and the screen decides which message to show.

### Data Files

| File | Record format |
| --- | --- |
| `Clients.txt` | `FirstName#//#LastName#//#Email#//#Phone#//#AccountNumber#//#PinCode#//#Balance` |
| `Users.txt` | `FirstName#//#LastName#//#Email#//#Phone#//#UserName#//#Password(shifted)#//#Permissions` |
| `Currencies.txt` | `Country#//#Code#//#Name#//#Rate (per 1 USD)` — 200+ countries / territories |
| `LoginRegister.txt` | `DateTime#//#UserName#//#Password(shifted)#//#Permissions` |
| `TransferLog.txt` | `DateTime#//#SourceAcc#//#DestAcc#//#Amount#//#SourceBalance#//#DestBalance#//#UserName` |

---

## Domain Model

| Class | Description |
| --- | --- |
| `InterfaceCommunication` | Abstract interface with `SendEmail`, `SendSMS`, `SendFax`. Defines the communication contract for a person. |
| `clsPerson` | Base class: first name, last name, email, phone, `FullName()`, `Print()`. Implements `InterfaceCommunication` (method bodies prepared as stubs). |
| `clsBankClient` | A bank account holder. Account number (unique), PIN code, balance. Find / Add / Update / Delete, `Deposit`, `Withdraw`, `Transfer`, total balances, and the transfer log. |
| `clsUser` | A system operator. Unique username, password, permissions bitmask, login registration. The `Admin` account is protected. |
| `clsCurrency` | Country, code, name, rate against 1 USD. Find by country/code, update rate (persisted immediately), and conversion methods. |
| `clsScreen` | Base class of all screens: draws the header (date, current user), and `_HasPermission()` shows *Access Denied* when needed. |
| `clsInputValidate<T>` | Template class for safe input: `ReadNumber`, `ReadNumberBetween`, `ReadPositiveNumber`, `ReadString`, `ReadChar`, date validation. |
| `clsString` | String utilities: split, join, trim, case conversion, encrypt / decrypt, and more. |
| `clsDate` | Date library: system date/time string, date arithmetic, comparisons, calendars. |
| `clsUtil` | Utilities: random generation, `NumberToText` (e.g. total balances in words), encrypt / decrypt. |
| `Global.h` | Holds `CurrentUser`, the logged-in user shared by all screens. |

---

## ✨ Features & Services

### Client Management

| Feature | Description |
| --- | --- |
| Show Client List | Table of all clients with a count sub-title |
| Add New Client | Account number must be unique, then reads the client data and prints a client card |
| Delete Client | Finds the client, shows the card, asks for confirmation, then removes it |
| Update Client | Finds the client, re-reads the data and saves it |
| Find Client | Search by account number and show the client card |

### Transactions

| Feature | Description |
| --- | --- |
| Deposit | Positive amount only, confirmation before saving |
| Withdraw | Amount cannot exceed the current balance |
| Total Balances | Table of all balances, the total, and **the total written in words** |
| Transfer | Source → destination, amount limited by the source balance, confirmation, and automatic log entry |
| Transfer Log | Full history of transfers with the user who made each one |

### Currency Exchange

| Feature | Description |
| --- | --- |
| List Currencies | All currencies and their rates against 1 USD |
| Find Currency | Search by **code** or **country** (case-insensitive) |
| Update Rate | Updates the rate and saves it to the file |
| Currency Calculator | Converts between any two currencies through USD, in a loop |

```
Amount in Currency1 ──► ÷ Rate1 ──► USD ──► × Rate2 ──► Amount in Currency2
```

### Users, Permissions & Security

- **Login** with a maximum of **3 attempts**.
- **Login Register**: every successful login is logged with date/time, username and permissions.
- **Manage Users**: list, add, delete, update, find.
- **Admin protection**: the `Admin` user cannot be added again, updated, or deleted.
- **Screen-level access control**: each screen calls `_HasPermission()` and shows *Access Denied! Contact your Admin.* when the user lacks the permission.

Permissions are stored as a **bitmask integer**. `-1` means full access, and any other value is a sum of the bits below:

| Permission | Value |
| --- | --- |
| `ShowClientList_p` | 1 |
| `AddNewClient_p` | 2 |
| `DeleteClient_p` | 4 |
| `UpdateClientInfo_p` | 8 |
| `FindClient_p` | 16 |
| `Transactions_p` | 32 |
| `CurrencyExchange_p` | 64 |
| `ManageUsers_p` | 128 |
| `LoginRegister_p` | 256 |
| `All_p` | -1 |

```cpp
bool CheckAccessPermission(enPermissions Permission)
{
    if (this->Permissions == enPermissions::All_p)
        return true;

    return (this->Permissions & short(Permission)) == Permission;
}
```

When the admin answers *Yes* to every permission question (sum = `511`), it is stored as `-1` (full access).

---

## Tech Stack

| Technology | Usage |
| --- | --- |
| C++ | Core language |
| OOP | Classes, inheritance, interfaces, encapsulation, static members, templates |
| STL | `string`, `vector`, `fstream`, `iomanip`, `random` |
| Text files | Persistent storage (`#//#` separated records) |
| MSVC extensions | `__declspec(property)` for C#-style properties |
| Windows console | `system("cls")`, `system("pause")` |

No external libraries are used.

---

## Project Structure

```
Bank-System/
│
├── main.cpp                           # Entry point: login loop
│
├── Global.h                           # CurrentUser (logged-in user)
├── InterfaceCommunication.h           # Abstract interface
│
├── clsPerson.h                        # Base class
├── clsBankClient.h                    # Inherits clsPerson
├── clsUser.h                          # Inherits clsPerson (permissions system)
├── clsCurrency.h
│
├── clsInputValidate.h                 # Template input validation
├── clsString.h
├── clsDate.h
├── clsUtil.h
│
├── clsScreen.h                        # Base class for all screens
├── clsLoginScreen.h
├── clsMainScreen.h
├── clsExitScreen.h
│
├── clsClientListScreen.h
├── clsAddNewClientScreen.h
├── clsDeleteClientScreen.h
├── clsUpdateClientScreen.h
├── clsFindClientScreen.h
│
├── clsTransactionsScreen.h            # Transactions menu
├── clsDepositScreen.h
├── clsWithdrawScreen.h
├── clsTotalBalancesScreen.h
├── clsTransferScreen.h
├── clsTransferLogListScreen.h
│
├── clsCurrencyExchangeScreen.h        # Currency menu
├── clsCurrenciesListScreen.h
├── clsFindCurrencyScreen.h
├── clsUpdateCurrencyRateScreen.h
├── clsCurrencyCalculatorScreen.h
│
├── clsManageUsersScreen.h             # Users menu
├── clsUsersListScreen.h
├── clsAddNewUserScreen.h
├── clsDeleteUserScreen.h
├── clsUpdateUserScreen.h
├── clsFindUserScreen.h
├── clsLoginRegisterScreen.h
│
├── Clients.txt                        # Clients database
├── Users.txt                          # Users database
├── Currencies.txt                     # Currencies database
├── LoginRegister.txt                  # Login audit log
└── TransferLog.txt                    # Transfer audit log
```

---

## 🚀 Getting Started

### Prerequisites

- Windows OS (the app uses `system("cls")` and `system("pause")`)
- Visual Studio 2019 or later with the **Desktop development with C++** workload (MSVC is required for `__declspec(property)`)

### Setup

1. **Clone the repository**

```
git clone https://github.com/YahYa-mzn/Bank-System.git
```

2. **Open the solution** — double-click the `.sln` file in Visual Studio.

3. **Run** — build and run (`F5` / `Ctrl + F5`). The data files (`Users.txt`, `Clients.txt`, `Currencies.txt`) are included, and `LoginRegister.txt` / `TransferLog.txt` are created automatically on first use.

4. **Log in** with the default credentials: `Admin` / `1234`

> In `Users.txt` passwords are stored shifted by a key of 3, so the stored value of `1234` looks like `4567`.

---

## ⚠️ Known Limitations & Planned Improvements

This was my first OOP project, and reviewing it today highlights what I would do differently:

- **Password security** — Passwords use a simple shift cipher (`clsString::Encrypt` with key 3), not hashing. Client PINs are stored as plain text, and some screens (Login Register, cards) display them.
- **Transfer to the same account** — Not blocked. Because the source and destination are separate objects, the destination deposit overwrites the withdrawal and the balance grows. A check `Source != Destination` is needed.
- **Balance precision** — Balances are `double` in the class, but the constructor takes a `float` and the file is parsed with `stof`, so large balances can lose precision. Should be `double` / `stod` end to end.
- **Login Register permission** — `LoginRegister_p` exists in the enum, but `clsLoginRegisterScreen` does not call `_HasPermission()` yet.
- **Recursive menu navigation** — Menus return by calling the menu again (`ShowMainMenu()` inside `_GoBackToMainMenuScreen()`), which grows the call stack. A loop would be cleaner.
- **Whole-file rewrite on every update** — Fine for learning, but not scalable and not safe for multi-user access (no locking or transactions; a transfer is two separate saves).
- **Currency rates per country row** — A rate update affects only the matching country row, not every country sharing the same code (e.g. EUR), and `France` appears twice in `Currencies.txt`.
- **Utility library leftovers** — A few methods in the course utility classes that this app never calls have known issues (e.g. `clsDate::NumberOfDaysInAYear` returns 364 for non-leap years, `NumberOfSecondsInAYear()` recurses forever, `clsString::TrimRight` on an all-spaces string, `clsUser::IsValid` missing a return on one path).
- **Communication interface** — `SendEmail`, `SendSMS` and `SendFax` are empty stubs, ready for a real implementation.
- **Windows / MSVC only** — Because of `__declspec(property)` and `system("cls")`.

**Planned / next steps:**

- Hash passwords and PINs
- Replace text files with a real database (the direction taken in my C# + SQL Server projects that followed)

---

## Credits

This project is based on **Course 11 by Mohammed Abu Hadhoud** from [Programming Advices](https://www.youtube.com/@ProgrammingAdvices).

Built by **Yahya Mazini** — [GitHub](https://github.com/YahYa-mzn)
