# DevOps Senior Prep

Система подготовки к интервью и прокачки навыков. Охватывает весь современный DevOps/SRE стек с акцентом на внутреннее устройство.

## Структура

| Директория | Тема | Статус |
|------------|------|--------|
| [linux/](linux/) | Linux internals | 🔨 в процессе |
| [kubernetes/](kubernetes/) | Архитектура и внутренности K8s | 🔨 в процессе |
| [networking/](networking/) | TCP/IP, DNS, iptables, service mesh | ⬜ todo |
| [observability/](observability/) | Prometheus, Grafana, трейсинг, SLO | ⬜ todo |
| [system-design/](system-design/) | System design в SRE-стиле | ⬜ todo |
| [security/](security/) | Zero trust, секреты, supply chain | ⬜ todo |
| [iac/](iac/) | Terraform, Ansible | ⬜ todo |
| [cicd/](cicd/) | GitHub Actions, ArgoCD, GitOps | ⬜ todo |
| [cloud/](cloud/) | AWS + GCP архитектура | ⬜ todo |
| [databases/](databases/) | PostgreSQL, Redis — операционные вопросы | ⬜ todo |

## Индекс вопросов для интервью

В каждой директории есть `interviews.md` с полными ответами:

- [Linux](linux/interviews.md)
- [Kubernetes](kubernetes/interviews.md)
- [Сети](networking/interviews.md)
- [System design](system-design/interviews.md)
- [Observability](observability/interviews.md)

## Главный инсайт для senior-интервью (2025-2026)

Senior-интервьюеры не проверяют знание инструментов — они проверяют **рассуждение в условиях отказа**.
Каждый ответ должен покрывать: как работает изнутри, failure modes, blast radius.

**Правило mitigation-first**: в любом инцидент-сценарии начинай с "первым делом снижу impact для пользователей..." — и только потом root cause.
