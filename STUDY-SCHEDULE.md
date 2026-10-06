# 🗓️ 35-Day Study Schedule (5 Weeks)

A realistic day-by-day plan assuming **2–4 focused hours/day**. Each day: **read the module → retype every example → do the mini-exercises at the end → note questions**. Weekends consolidate and practice.

> Tip: Keep a single `scratch/` folder of `.php` files and a throwaway Laravel app (`demo-app`) where you reproduce every example yourself.

---

## Week 1 — PHP Fundamentals
| Day | Focus | Modules |
|-----|-------|---------|
| 1 | Setup + how PHP runs | [PHP 01](php/01-introduction-and-setup.md) |
| 2 | Variables, types, type juggling | [PHP 02](php/02-syntax-variables-datatypes.md) |
| 3 | Operators (incl. `??`, `<=>`, `match`) | [PHP 03](php/03-operators.md) |
| 4 | Control structures + loops | [PHP 04](php/04-control-structures.md), [PHP 05](php/05-loops.md) |
| 5 | Functions, closures, arrow fns | [PHP 06](php/06-functions-and-closures.md) |
| 6 | Strings (deep dive) | [PHP 07](php/07-strings.md) |
| 7 | Arrays (deep dive) + **review week 1** | [PHP 08](php/08-arrays.md) |

## Week 2 — PHP OOP & Core
| Day | Focus | Modules |
|-----|-------|---------|
| 8 | Forms, superglobals, sessions | [PHP 09](php/09-superglobals-forms-sessions.md) |
| 9 | OOP fundamentals | [PHP 10](php/10-oop-fundamentals.md) |
| 10 | Inheritance, interfaces, traits | [PHP 11](php/11-oop-advanced-inheritance-interfaces-traits.md) |
| 11 | Magic methods | [PHP 12](php/12-magic-methods.md) |
| 12 | Namespaces & autoloading | [PHP 13](php/13-namespaces-and-autoloading.md) |
| 13 | Errors & exceptions | [PHP 14](php/14-error-and-exception-handling.md) |
| 14 | Type system & enums + **review** | [PHP 15](php/15-type-system-and-enums.md) |

## Week 3 — Advanced PHP → Laravel start
| Day | Focus | Modules |
|-----|-------|---------|
| 15 | Dates, files, JSON | [PHP 16](php/16-dates-files-json.md) |
| 16 | Databases: PDO | [PHP 17](php/17-database-pdo-mysqli.md) |
| 17 | Composer & PSR | [PHP 18](php/18-composer-and-psr-standards.md) |
| 18 | Generators, SPL, regex | [PHP 19](php/19-advanced-generators-spl-regex.md) |
| 19 | Security + design patterns | [PHP 20](php/20-security-best-practices.md), [PHP 21](php/21-design-patterns-and-solid.md) |
| 20 | Laravel intro + install | [L01](laravel/01-introduction-and-installation.md) |
| 21 | Request lifecycle + **PHP interview Q review** | [L02](laravel/02-request-lifecycle.md), [PHP Qs](interview/php-interview-questions.md) |

## Week 4 — Laravel Core
| Day | Focus | Modules |
|-----|-------|---------|
| 22 | Routing + controllers | [L03](laravel/03-routing.md), [L04](laravel/04-controllers.md) |
| 23 | Middleware + req/resp | [L05](laravel/05-middleware.md), [L06](laravel/06-requests-and-responses.md) |
| 24 | Blade | [L07](laravel/07-blade-templating.md) |
| 25 | Validation | [L08](laravel/08-validation.md) |
| 26 | Eloquent basics | [L09](laravel/09-eloquent-basics.md) |
| 27 | Eloquent relationships (N+1!) | [L10](laravel/10-eloquent-relationships.md) |
| 28 | Migrations/seeders/factories + query builder + collections + container/facades + **review** | [L11](laravel/11-migrations-seeders-factories.md), [L12](laravel/12-query-builder-and-database.md), [L13](laravel/13-collections.md), [L14](laravel/14-service-container-and-providers.md), [L15](laravel/15-facades-and-contracts.md) |

## Week 5 — Advanced Laravel + Interview Prep
| Day | Focus | Modules |
|-----|-------|---------|
| 29 | Auth + authorization | [L16](laravel/16-authentication.md), [L17](laravel/17-authorization-gates-policies.md) |
| 30 | API resources + API/Sanctum | [L18](laravel/18-api-resources-and-serialization.md), [L23](laravel/23-api-development-and-sanctum.md) |
| 31 | Events, queues, scheduling | [L19](laravel/19-events-and-listeners.md), [L20](laravel/20-queues-and-jobs.md), [L21](laravel/21-task-scheduling.md) |
| 32 | Mail/notifications, caching, storage | [L22](laravel/22-mail-and-notifications.md), [L24](laravel/24-caching-and-sessions.md), [L25](laravel/25-file-storage.md) |
| 33 | Testing + Artisan + architecture | [L26](laravel/26-testing.md), [L27](laravel/27-artisan-and-custom-commands.md), [L29](laravel/29-architecture-patterns-in-laravel.md) |
| 34 | Localization/deploy/optimize + **build a project** | [L28](laravel/28-localization-packages-deployment-optimization.md), [Projects](projects/practice-projects.md) |
| 35 | **Full interview review + mock** | [Laravel Qs](interview/laravel-interview-questions.md), [Coding challenges](interview/coding-challenges.md) |

---

## Daily habits that build mastery
- **Explain out loud**: after each module, explain the concept as if teaching someone. If you can't, re-read.
- **Flashcards**: turn every 🎯 Interview Tip into a Q→A flashcard.
- **One bug a day**: deliberately break an example and predict the error before running it.
- **Read source**: in Week 4–5, open `vendor/laravel/framework` and read the class behind a feature you just learned.
