# 🚀 PHP & Laravel Mastery — Zero to Expert in 5 Weeks

A complete, example-driven course designed to take you from the absolute basics of PHP all the way to advanced Laravel and interview readiness. Every concept is documented with explanations, syntax, runnable examples, expected output, common mistakes, best practices, and interview tips.

> **Target versions:** PHP 8.4 (notes for 8.1–8.3) and Laravel 12 (notes for Laravel 10/11).
> **Goal:** Be able to confidently answer *any* PHP/Laravel interview question and write production-quality code.

---

## 📚 How to use this course

1. **Follow the 5-week plan below** (or the granular [day-by-day schedule](STUDY-SCHEDULE.md)).
2. For each module: **read → type out every example yourself → break it → fix it**. Do not copy-paste.
3. End each module by reviewing its **🎯 Interview Tips** and **Quick Reference** sections.
4. Each Friday, do the relevant **coding challenges** and review the **interview question banks**.
5. In Week 5, build at least one **practice project** end-to-end.

### Setting up your environment
```bash
# macOS (Homebrew)
brew install php composer
php -v          # expect PHP 8.4.x
composer -V

# Verify a quick script
echo '<?php echo "Hello, PHP " . PHP_VERSION . PHP_EOL;' > hello.php
php hello.php

# Laravel (after PHP + Composer are installed)
composer global require laravel/installer
laravel new demo-app        # or: composer create-project laravel/laravel demo-app
cd demo-app && php artisan serve
```
A database (MySQL/MariaDB or SQLite) and an IDE (VS Code + PHP Intelephense, or PhpStorm) are recommended.

---

## 🗓️ The 5-Week Plan

### Week 1 — PHP Fundamentals
| # | Module |
|---|--------|
| 01 | [Introduction & Setup](php/01-introduction-and-setup.md) |
| 02 | [Syntax, Variables & Data Types](php/02-syntax-variables-datatypes.md) |
| 03 | [Operators](php/03-operators.md) |
| 04 | [Control Structures](php/04-control-structures.md) |
| 05 | [Loops](php/05-loops.md) |
| 06 | [Functions & Closures](php/06-functions-and-closures.md) |
| 07 | [Strings](php/07-strings.md) |
| 08 | [Arrays](php/08-arrays.md) |

### Week 2 — PHP OOP & Core
| # | Module |
|---|--------|
| 09 | [Superglobals, Forms & Sessions](php/09-superglobals-forms-sessions.md) |
| 10 | [OOP Fundamentals](php/10-oop-fundamentals.md) |
| 11 | [OOP Advanced: Inheritance, Interfaces & Traits](php/11-oop-advanced-inheritance-interfaces-traits.md) |
| 12 | [Magic Methods](php/12-magic-methods.md) |
| 13 | [Namespaces & Autoloading](php/13-namespaces-and-autoloading.md) |
| 14 | [Error & Exception Handling](php/14-error-and-exception-handling.md) |
| 15 | [Type System & Enums](php/15-type-system-and-enums.md) |

### Week 3 — Advanced PHP → Laravel Foundations
| # | Module |
|---|--------|
| 16 | [Dates, Files & JSON](php/16-dates-files-json.md) |
| 17 | [Databases: PDO & MySQLi](php/17-database-pdo-mysqli.md) |
| 18 | [Composer & PSR Standards](php/18-composer-and-psr-standards.md) |
| 19 | [Advanced PHP: Generators, SPL & Regex](php/19-advanced-generators-spl-regex.md) |
| 20 | [Security Best Practices](php/20-security-best-practices.md) |
| 21 | [Design Patterns & SOLID](php/21-design-patterns-and-solid.md) |
| L01 | [Laravel: Introduction & Installation](laravel/01-introduction-and-installation.md) |
| L02 | [Laravel: Request Lifecycle](laravel/02-request-lifecycle.md) |

### Week 4 — Laravel Core
| # | Module |
|---|--------|
| L03 | [Routing](laravel/03-routing.md) |
| L04 | [Controllers](laravel/04-controllers.md) |
| L05 | [Middleware](laravel/05-middleware.md) |
| L06 | [Requests & Responses](laravel/06-requests-and-responses.md) |
| L07 | [Blade Templating](laravel/07-blade-templating.md) |
| L08 | [Validation](laravel/08-validation.md) |
| L09 | [Eloquent Basics](laravel/09-eloquent-basics.md) |
| L10 | [Eloquent Relationships](laravel/10-eloquent-relationships.md) |
| L11 | [Migrations, Seeders & Factories](laravel/11-migrations-seeders-factories.md) |
| L12 | [Query Builder & Database](laravel/12-query-builder-and-database.md) |
| L13 | [Collections](laravel/13-collections.md) |
| L14 | [Service Container & Providers](laravel/14-service-container-and-providers.md) |
| L15 | [Facades & Contracts](laravel/15-facades-and-contracts.md) |

### Week 5 — Advanced Laravel + Interview Prep
| # | Module |
|---|--------|
| L16 | [Authentication](laravel/16-authentication.md) |
| L17 | [Authorization: Gates & Policies](laravel/17-authorization-gates-policies.md) |
| L18 | [API Resources & Serialization](laravel/18-api-resources-and-serialization.md) |
| L19 | [Events & Listeners](laravel/19-events-and-listeners.md) |
| L20 | [Queues & Jobs](laravel/20-queues-and-jobs.md) |
| L21 | [Task Scheduling](laravel/21-task-scheduling.md) |
| L22 | [Mail & Notifications](laravel/22-mail-and-notifications.md) |
| L23 | [API Development & Sanctum](laravel/23-api-development-and-sanctum.md) |
| L24 | [Caching & Sessions](laravel/24-caching-and-sessions.md) |
| L25 | [File Storage](laravel/25-file-storage.md) |
| L26 | [Testing (PHPUnit & Pest)](laravel/26-testing.md) |
| L27 | [Artisan & Custom Commands](laravel/27-artisan-and-custom-commands.md) |
| L28 | [Localization, Packages, Deployment & Optimization](laravel/28-localization-packages-deployment-optimization.md) |
| L29 | [Architecture Patterns in Laravel](laravel/29-architecture-patterns-in-laravel.md) |

### Interview & Practice (use throughout, focus in Week 5)
- 🎤 [PHP Interview Questions & Answers](interview/php-interview-questions.md)
- 🎤 [Laravel Interview Questions & Answers](interview/laravel-interview-questions.md)
- 🧩 [Coding Challenges (with solutions)](interview/coding-challenges.md)
- 🏗️ [Practice Projects](projects/practice-projects.md)

---

## ✅ Mastery checklist
By the end you should be able to, without notes:
- Explain PHP's type system, references, and how arrays/strings work internally.
- Write clean OOP code using interfaces, traits, abstract classes, and the right design pattern.
- Explain the Laravel request lifecycle and the service container end-to-end.
- Model any data relationship in Eloquent and avoid N+1 queries.
- Build a secure, validated, authenticated REST API with tests.
- Reason about queues, events, caching, and deployment trade-offs.
- Answer "how does X work under the hood?" for the common interview topics.

Good luck — and remember: **the difference between knowing and mastering is typing the code yourself.**
