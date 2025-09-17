### **Architecture Document: A Guide to the Modular Monolith**

#### **1. Philosophy and Guiding Principles**

The objective of this architecture is to build a robust, scalable, and maintainable application by organizing code around clear business logic. We design the system as a set of **business modules with explicit boundaries**, where each module represents a distinct capability of the application.

- **Guiding Principles:**

  - **Modularity by Business Capability:** The code is grouped by functionality (Vertical Slicing). Everything related to "Invoicing" is encapsulated within the `Invoicing` module.
  - **Explicit Boundaries (Public API):** Modules **never** communicate directly with the implementations of others. They expose a public API through `Contracts` (for synchronous communication) and `Events` (for asynchronous communication).
  - **Standardized Communication via DTOs:** The `Data Transfer Object` (`spatie/laravel-data`) is the sole vehicle for our data exchanges. It guarantees the **structure**, **validation**, and **transport** of information.
  - **Immutability Preferred:** Objects that carry data (`DTOs`, `Events`, `Value Objects`) are systematically `readonly` to ensure predictable and side-effect-free data flows.
  - **Explicit Dependencies:** "Magic" is avoided. Dependencies are injected. The use of Laravel facades is strictly controlled.
  - **Strict Naming Conventions:** Each business class is suffixed with its type (`...Action`, `...Query`, `...Service`, `...Contract`, `...DTO`) for immediate readability and code searchability.

- **Non-Goals:**
  - This architecture does **not** aim to create distributable microservices. It accepts the trade-off of a shared database to maintain the operational simplicity of a monolith.
  - Raw performance is **not** the absolute priority over clarity, testability, and long-term maintainability.

---

#### **2. Glossary of Key Concepts**

- **Modular Monolith:** An architecture where the application is deployed as a single unit, but its source code is organized into independent and loosely coupled modules.
- **Module's Public API:** The set of `Contracts` (interfaces) and `Events` that a module exposes to the rest of the application. It is the only authorized contact surface.
- **Lightweight CQRS:** The strict separation of responsibilities between **read** operations (`Queries`) and **write** operations (`Actions`).
- **Action:** A class with a single `execute()` method that performs a single write operation (creation, modification, deletion). It contains the internal business logic of a module.
- **Query:** A class with a single `execute()` method that is responsible for retrieving data. It can use inter-module Eloquent relationships but **must** return DTOs.
- **Service:** The implementation of a `Contract`. It serves as a public facade for a module. It contains **no business logic** and merely delegates calls to the module's internal `Actions` or `Queries`.
- **DTO (Data Transfer Object):** A `readonly` object (`spatie/laravel-data`) whose sole purpose is to transport structured data.
- **Value Object (VO):** A small `readonly` object that represents a simple domain concept (e.g., `Money`, `EmailAddress`). It validates its own integrity upon creation.
- **ViewModel:** An object that prepares domain data for a specific display (e.g., a Blade view, a JSON response).

---

#### **3. The Core Stack (Packages)**

The selection of these packages provides a coherent, high-quality foundation. Their use is considered a standard for any new project.

- **Quality & Architecture:**
  - `larastan/larastan`: Static analysis to prevent type errors.
  - `pestphp/pest` & `pestphp/pest-plugin-arch`: Testing framework and a plugin to validate our architectural rules automatically.
  - `laravel/pint`: Automatic and uniform code formatting.
- **Data Structuring:**
  - `spatie/laravel-data`: Standardizes DTO management, validation, and transformation.
- **Operations & Reliability:**
  - `spatie/laravel-backup`: A reliable solution for backups.
  - `spatie/laravel-health`: A dashboard for application monitoring.
- **Common Features:**
  - `spatie/laravel-medialibrary`: Robust media management.
  - `spatie/laravel-permission`: The standard for managing roles and permissions.

---

#### **4. Detailed File Structure**

This file tree is the concrete and complete representation of our modular approach.

```
.
├── src/
│   ├── Modules/
│   │   ├── Invoicing/
│   │   │   ├── Actions/
│   │   │   │   └── CreateInvoiceAction.php
│   │   │   ├── Contracts/
│   │   │   │   └── InvoicingServiceContract.php
│   │   │   ├── Database/
│   │   │   │   ├── Factories/
│   │   │   │   └── Migrations/
│   │   │   ├── Data/
│   │   │   │   └── InvoiceData.php
│   │   │   ├── Enums/
│   │   │   ├── Events/
│   │   │   │   └── InvoiceCreatedEvent.php
│   │   │   ├── Http/
│   │   │   │   ├── Controllers/
│   │   │   │   └── ViewModels/
│   │   │   ├── Listeners/
│   │   │   ├── Models/
│   │   │   ├── Policies/
│   │   │   ├── Providers/
│   │   │   │   └── InvoicingServiceProvider.php
│   │   │   ├── Queries/
│   │   │   │   └── GetInvoiceDetailsQuery.php
│   │   │   ├── routes/
│   │   │   │   ├── api.php
│   │   │   │   └── web.php
│   │   │   ├── Services/
│   │   │   │   └── InvoicingService.php
│   │   │   ├── Tests/
│   │   │   │   ├── Actions/
│   │   │   │   └── Models/
│   │   │   ├── ValueObjects/
│   │   │   │   └── InvoiceNumber.php
│   │   │   ├── config.php
│   │   │   └── README.md
│   │   └── ...
│   └── Shared/
│       ├── Casts/
│       ├── Validation/
│       └── ValueObjects/
├── tests/
│   ├── Architecture/
│   │   └── Dependencies.php
│   ├── Feature/
│   │   └── Invoicing/
│   ├── Modules/
│   │   └── Invoicing/
│   └── Unit/
├── bootstrap/
│   └── providers.php
└── composer.json
```

---

#### **5. The Pillars of the Architecture: Explanation of Folders**

- **`Providers/`: The Heart of the Module**
  The main `ServiceProvider` binds the public interface (`Contract`) to its implementation (`Service`). It is also responsible for registering routes, migrations, configurations, policies, etc.

- **`Contracts/` and `Services/`: The Synchronous Public API**

  - **`Contracts/`** contains the interfaces (`...Contract.php`). **This is the only way for another module to communicate synchronously with this one.**
  - **`Services/`** contains the implementations (`...Service.php`). A `Service` class is a simple facade: it contains **no business logic**. Its sole role is to delegate calls to internal `Actions` or `Queries`.

- **`Actions/` and `Queries/`: The Internal Business Logic**

  - **`Actions/` (Write)**: Single-responsibility classes that modify the system's state.
  - **`Queries/` (Read)**: Single-responsibility classes that retrieve data. This is the only place where inter-module relationships can be used (see section 6.3).

- **`Http/Controllers/`: The Orchestra Conductor**
  Its only role is to: 1) Validate the input (via a DTO), 2) Call a single `Action` or `Query`, 3) Return a response (often via a `ViewModel`). It contains no business logic.

- **`Models/` and `Policies/`: Data and its Security**

  - **`Models/`** contains the module's Eloquent models.
  - **`Policies/`** contains the authorization logic related to the models. They are registered in the module's `ServiceProvider`.

- **`Events/` and `Listeners/`: Asynchronous Communication**
  - **`Events/`** defines the business events emitted by the module (via a DTO).
  - **`Listeners/`** reacts to events. **Any Listener that reacts to an event from another module MUST implement the `ShouldQueue` interface** to ensure decoupling and resilience.

---

#### **6. Cross-Cutting Rules and Key Policies**

##### **6.1. The `Shared/` Folder Rule**

This folder is for **technical and universal** code, 100% agnostic of any business context. Before adding code here, ask the question: "Could this code be published as an open-source package independent of my application?". If the answer is no, its place is in a module.

##### **6.2. Laravel Facades Usage Rule**

- **Allowed (Infrastructure):** `Route`, `Schema`, `Log`, `Cache`, `Config`, `Storage`, `Gate`, `DB`.
- **Forbidden in Business Logic (`Actions`, `Queries`, `Services`, `Models`):** `Auth`, `Request`, `Session`. These dependencies must be injected explicitly.

##### **6.3. The Exception Rule: Controlled Use of Inter-Module Relations**

To combine productivity and decoupling, the use of Eloquent relationships that cross module boundaries is tolerated under a strictly controlled exception policy.

**Fundamental Rule:**

> The one and only place where an Eloquent relationship pointing to another module's model can be loaded and used is inside a `Query` class. A `Query` that loads inter-module data **MUST NOT** return the raw Eloquent models. It **MUST** convert the result into a collection of DTOs before returning it.

    ```php
    // In src/Modules/Blog/Queries/GetPostsQuery.php
    public function execute(): \Spatie\LaravelData\DataCollection
    {
        $posts = Post::with('author')->get();
        return PostWithAuthorData::collection($posts);
    }
    ```

---

#### **7. Four-Level Testing Strategy**

1.  **Level 1: Architecture Tests (Pest Arch)**

    - **Location:** `tests/Architecture/`.
    - **Purpose:** To validate that our architectural rules are being followed. It is the automated guardian of the architecture.

2.  **Level 2: Unit Tests (Co-located)**

    - **Location:** `src/Modules/Invoicing/Tests/`.
    - **Purpose:** To test a class in complete isolation (mocked dependencies).

3.  **Level 3: Module API Tests**

    - **Location:** `tests/Modules/Invoicing/`.
    - **Purpose:** To test a module's public API by interacting with its `Contracts`.

4.  **Level 4: Integration Tests (Feature Tests)**
    - **Location:** `tests/Feature/Invoicing/`.
    - **Purpose:** To test a complete flow via a simulated HTTP request.

---

#### **8. Practical Workflow: Adding a New "Orders" Module**

1.  **Create the structure:** Create the `src/Modules/Orders` folder with the entire required directory tree.
2.  **Update `composer.json`:** Add the `Modules\\` namespace to the PSR-4 autoload section and run `composer dump-autoload`.
3.  **Create the `ServiceProvider`:** Create `OrdersServiceProvider.php` and fill in the `boot()` and `register()` methods.
4.  **Register the Provider:** Add `Modules\Orders\Providers\OrdersServiceProvider::class` to `bootstrap/providers.php`.
5.  **Configure static analysis:** Add the new module's path to `phpstan.neon`.
6.  **Create documentation:** Create a `src/Modules/Orders/README.md` file.
7.  **Develop:** Create your models, DTOs, actions, controllers, and tests following the established principles.

---

#### **9. Documentation and Module Discovery**

Each module **must** have a `README.md` file at its root (`src/Modules/Invoicing/README.md`) that documents its public API:

- The module's responsibility.
- A list of its public `Contracts` and their usage.
- A list of the `Events` it emits, along with the structure of their DTOs.

---

#### **10. Warning: Residual Database Coupling**

This architecture does not resolve **coupling at the database level**. This is its fundamental trade-off. To manage it, the following rules are mandatory:

- **No inter-module `JOIN`s in raw queries.** Data composition must be done at the application level by calling other modules' `Queries` or by using the relationship exception rule.
- **A module never directly modifies a table belonging to another module.** All modifications must go through an `Action` of the owner module.
