# Паттерны проектирования — Business Metrics Analyzer

## 1. Таблица паттернов

| Паттерн | Где применяется | Обоснование | Альтернативы |
|---|---|---|---|
| **Repository** | Слой доступа к данным о продажах, пользователях | Изоляция бизнес-логики от SQLAlchemy, упрощает тестирование (mock-репозитории) | Прямые запросы к ORM в сервисах |
| **Unit of Work** | Импорт данных, транзакции KPI | Атомарность операций импорта, откат при ошибках валидации | Ручное управление `session.commit()` |
| **Service Layer** | Расчёт KPI, импорт, отчёты | Инкапсуляция бизнес-сценариев, переиспользование из API и CLI | Логика в контроллерах FastAPI |
| **Factory** | Создание отчётов (Excel, PDF) | Гибкое расширение типов отчётов без правки клиентского кода | `if/elif` по типу отчёта |
| **Strategy** | Различные методы расчёта KPI (рентабельность, средний чек, конверсия) | Взаимозаменяемость алгоритмов, соответствие OCP | Жёстко заданные методы в KpiService |
| **DTO** | Передача данных через REST API | Чёткие контракты, версионирование API, валидация Pydantic | Возврат сырых ORM-моделей |
| **Adapter** | Интеграция с 1С, SMTP, S3 | Изоляция внешних зависимостей, лёгкая замена провайдера | Прямые вызовы библиотек в сервисах |
| **Observer** | Обновление дашборда при изменении данных | Реакция на события импорта, push-обновления | Периодический polling с клиента |

## 2. Фрагменты кода на Python

### 2.1. Repository + Unit of Work (слой Infrastructure)
```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from decimal import Decimal

@dataclass
class SaleRecord:
    id: int | None
    month: str
    revenue: Decimal
    cost: Decimal

class SaleRepository(ABC):
    @abstractmethod
    def add(self, record: SaleRecord) -> None: ...

    @abstractmethod
    def get_by_period(self, period: str) -> list[SaleRecord]: ...

class SqlSaleRepository(SaleRepository):
    def __init__(self, session):
        self.session = session

    def add(self, record: SaleRecord) -> None:
        self.session.add(record)

    def get_by_period(self, period: str) -> list[SaleRecord]:
        return (
            self.session.query(SaleRecord)
            .filter(SaleRecord.month == period)
            .all()
        )

class UnitOfWork:
    def __init__(self, session_factory):
        self.session_factory = session_factory
        self.session = None
        self.sales: SqlSaleRepository | None = None

    def __enter__(self):
        self.session = self.session_factory()
        self.sales = SqlSaleRepository(self.session)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            self.session.commit()
        else:
            self.session.rollback()
        self.session.close()

### 2.2. Strategy для расчёта KPI (слой Domain)
from abc import ABC, abstractmethod

class KpiStrategy(ABC):
    @abstractmethod
    def calculate(self, revenue: float, cost: float) -> float: ...

class ProfitabilityStrategy(KpiStrategy):
    def calculate(self, revenue: float, cost: float) -> float:
        return 0.0 if revenue == 0 else (revenue - cost) / revenue * 100

class AverageCheckStrategy(KpiStrategy):
    def calculate(self, revenue: float, orders: int) -> float:
        return 0.0 if orders == 0 else revenue / orders

class ConversionStrategy(KpiStrategy):
    def calculate(self, visitors: int, buyers: int) -> float:
        return 0.0 if visitors == 0 else buyers / visitors * 100

### 2.3. Factory для отчётов (слой Application)
from abc import ABC, abstractmethod

class ReportGenerator(ABC):
    @abstractmethod
    def generate(self, data: dict) -> bytes: ...

class ExcelReportGenerator(ReportGenerator):
    def generate(self, data: dict) -> bytes:
        import openpyxl, io
        wb = openpyxl.Workbook()
        ws = wb.active
        for row in data.get("rows", []):
            ws.append(row)
        buf = io.BytesIO()
        wb.save(buf)
        return buf.getvalue()

class PdfReportGenerator(ReportGenerator):
    def generate(self, data: dict) -> bytes:
        # Заглушка: реальная реализация через reportlab / weasyprint
        return b"%PDF-1.4 ..."

class ReportFactory:
    _registry = {"excel": ExcelReportGenerator, "pdf": PdfReportGenerator}

    @classmethod
    def create(cls, report_type: str) -> ReportGenerator:
        try:
            return cls._registry[report_type]()
        except KeyError:
            raise ValueError(f"Неизвестный тип отчёта: {report_type}")
            

        