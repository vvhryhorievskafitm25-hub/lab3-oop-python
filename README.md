
"""
Лабораторна робота №3
Тема: Реалізація обробки виключень, модульності та документації в Python.

Автор: Григор'євська Вікторія
Призначення: система обробки та перевірки даних студентів.
"""

from pathlib import Path
from dataclasses import dataclass


# ============================================================
# 1. ВЛАСНА ІЄРАРХІЯ ВИКЛЮЧЕНЬ
# ============================================================

class AppError(Exception):
    """Базове виключення програми."""

    def __init__(self, message: str):
        """Ініціалізує повідомлення про помилку."""
        self.message = message
        super().__init__(message)

    def __str__(self) -> str:
        """Повертає текст помилки."""
        return f"Помилка програми: {self.message}"


class ValidationError(AppError):
    """Помилка перевірки введених даних."""

    def __init__(self, field: str, value: object, message: str):
        """Зберігає поле, неправильне значення та повідомлення."""
        self.field = field
        self.value = value
        super().__init__(message)

    def __str__(self) -> str:
        """Формує повідомлення про помилку перевірки."""
        return (
            f"Помилка поля '{self.field}': {self.message}. "
            f"Отримане значення: {self.value!r}"
        )


class ResourceNotFoundError(AppError):
    """Помилка пошуку відсутнього студента."""

    def __init__(self, student_id: int):
        """Зберігає ідентифікатор студента."""
        self.student_id = student_id
        super().__init__(f"Студента з ID {student_id} не знайдено.")

    def __str__(self) -> str:
        """Повертає повідомлення про відсутній запис."""
        return self.message


class DataProcessingError(AppError):
    """Помилка обробки або збереження даних."""

    def __init__(self, source: str, message: str):
        """Зберігає джерело помилки."""
        self.source = source
        super().__init__(message)

    def __str__(self) -> str:
        """Повертає повідомлення про помилку обробки."""
        return f"Помилка обробки '{self.source}': {self.message}"


# ============================================================
# 2. МОДЕЛЬ СТУДЕНТА
# ============================================================

@dataclass
class Student:
    """Модель студента з основними навчальними даними."""

    student_id: int
    name: str
    average_grade: float

    def validate(self) -> None:
        """Перевіряє правильність даних студента.

        Raises:
            ValidationError: Якщо дані неправильні.
        """
        if self.student_id <= 0:
            raise ValidationError(
                "student_id", self.student_id,
                "Ідентифікатор має бути додатним"
            )

        if not self.name.strip():
            raise ValidationError(
                "name", self.name,
                "Ім'я не може бути порожнім"
            )

        if not 0 <= self.average_grade <= 100:
            raise ValidationError(
                "average_grade", self.average_grade,
                "Бал має бути від 0 до 100"
            )

    def describe(self) -> str:
        """Повертає текстову інформацію про студента."""
        return (
            f"ID: {self.student_id} | "
            f"Ім'я: {self.name} | "
            f"Середній бал: {self.average_grade:.2f}"
        )


# ============================================================
# 3. ФУНКЦІЇ ПЕРЕВІРКИ ДАНИХ
# ============================================================

def parse_average(value: object) -> float:
    """Перетворює значення середнього бала на число.

    Args:
        value: Значення для перевірки.

    Returns:
        Перевірений середній бал.

    Raises:
        ValidationError: Якщо значення некоректне.
    """
    try:
        result = float(value)
    except (ValueError, TypeError) as error:
        raise ValidationError(
            "average_grade",
            value,
            "Значення не можна перетворити на число"
        ) from error

    if not 0 <= result <= 100:
        raise ValidationError(
            "average_grade",
            value,
            "Бал має бути в межах від 0 до 100"
        )

    return result


def validate_name(name: str) -> str:
    """Перевіряє ім'я студента."""
    cleaned_name = name.strip()

    if not cleaned_name:
        raise ValidationError("name", name, "Ім'я порожнє")

    return cleaned_name


# ============================================================
# 4. ОСНОВНИЙ КЛАС ДЛЯ РОБОТИ ЗІ СТУДЕНТАМИ
# ============================================================

class StudentService:
    """Керує студентськими записами."""

    def __init__(self):
        """Створює порожній список студентів."""
        self.students: list[Student] = []

    def add_student(
        self,
        student_id: int,
        name: str,
        average_grade: object
    ) -> Student:
        """Додає студента після перевірки його даних.

        Args:
            student_id: Ідентифікатор студента.
            name: Ім'я студента.
            average_grade: Середній бал.

        Returns:
            Створений об'єкт Student.

        Raises:
            AppError: Якщо дані некоректні або ID повторюється.
        """
        if any(s.student_id == student_id for s in self.students):
            raise AppError(f"ID {student_id} вже існує.")

        student = Student(
            student_id,
            validate_name(name),
            parse_average(average_grade)
        )

        student.validate()
        self.students.append(student)
        return student

    def find_student(self, student_id: int) -> Student:
        """Шукає студента за ідентифікатором.

        Raises:
            ResourceNotFoundError: Якщо студента не знайдено.
        """
        for student in self.students:
            if student.student_id == student_id:
                return student

        raise ResourceNotFoundError(student_id)

    def save_to_file(self, filename: str) -> None:
        """Зберігає записи студентів у текстовий файл."""
        try:
            path = Path(filename)
            path.parent.mkdir(parents=True, exist_ok=True)

            with path.open("w", encoding="utf-8") as file:
                for student in self.students:
                    file.write(
                        f"{student.student_id};"
                        f"{student.name};"
                        f"{student.average_grade}\n"
                    )

        except OSError as error:
            raise DataProcessingError(
                filename,
                "Не вдалося зберегти дані"
            ) from error

    def add_with_logging(
        self,
        student_id: int,
        name: str,
        average_grade: object
    ) -> Student:
        """Додає студента та повторно генерує помилку після журналювання."""
        try:
            return self.add_student(
                student_id, name, average_grade
            )
        except AppError as error:
            print(f"[ЖУРНАЛ ПОМИЛОК] {error}")
            raise


# ============================================================
# 5. БАЗОВА ОБРОБКА ВИКЛЮЧЕНЬ
# ============================================================

def basic_exception_demo() -> None:
    """Демонструє використання try та except."""
    print("\n1. БАЗОВА ОБРОБКА ВИКЛЮЧЕНЬ")

    values = ["19", "не число"]

    for value in values:
        try:
            number = int(value)
            print("Результат:", number)

        except ValueError:
            print(f"Помилка: '{value}' не є цілим числом.")


# ============================================================
# 6. TRY, EXCEPT, ELSE, FINALLY ТА МЕНЕДЖЕР КОНТЕКСТУ
# ============================================================

def full_exception_demo() -> None:
    """Демонструє повну конструкцію обробки виключень."""
    print("\n2. TRY, EXCEPT, ELSE, FINALLY ТА WITH")

    filename = Path("data/students.txt")

    try:
        with filename.open("r", encoding="utf-8") as file:
            content = file.read()

    except FileNotFoundError:
        print("Файл не знайдено.")

    except OSError as error:
        print("Помилка роботи з файлом:", error)

    else:
        print("Файл успішно прочитано.")
        print(content)

    finally:
        print("Блок finally виконано.")


# ============================================================
# 7. ОБРОБКА КІЛЬКОХ ТИПІВ ВИКЛЮЧЕНЬ
# ============================================================

def multiple_exception_demo() -> None:
    """Демонструє перехоплення кількох типів виключень."""
    print("\n3. ОБРОБКА КІЛЬКОХ ВИКЛЮЧЕНЬ")

    values = ["25.5", "не число", None]

    for value in values:
        try:
            result = float(value)
            print("Результат:", result)

        except (ValueError, TypeError) as error:
            print(
                f"Перехоплено {type(error).__name__}: {error}"
            )


# ============================================================
# 8. ВЛАСНІ ВИКЛЮЧЕННЯ ТА RAISE FROM
# ============================================================

def custom_exception_demo() -> None:
    """Демонструє створення та перехоплення власних помилок."""
    print("\n4. ВЛАСНІ ВИКЛЮЧЕННЯ")

    try:
        raise ValidationError(
            "average_grade", 150,
            "Бал не може перевищувати 100"
        )
    except ValidationError as error:
        print("ValidationError:", error)

    try:
        raise ResourceNotFoundError(999)
    except ResourceNotFoundError as error:
        print("ResourceNotFoundError:", error)

    try:
        try:
            raise FileNotFoundError("Початковий файл відсутній")
        except FileNotFoundError as original:
            raise DataProcessingError(
                "students.txt",
                "Не вдалося обробити файл"
            ) from original

    except DataProcessingError as error:
        print("DataProcessingError:", error)
        print("Початкова причина:", repr(error.__cause__))

    try:
        raise ValidationError(
            "name", "",
            "Ім'я не може бути порожнім"
        )
    except AppError as error:
        print("Перехоплення базового класу:", error)


# ============================================================
# 9. ПОВТОРНЕ ГЕНЕРУВАННЯ ВИКЛЮЧЕННЯ
# ============================================================

def reraising_demo() -> None:
    """Демонструє повторне генерування виключення через raise."""
    print("\n5. ПОВТОРНЕ ГЕНЕРУВАННЯ")

    service = StudentService()

    try:
        service.add_with_logging(1, "", 90)

    except AppError as error:
        print("Помилку повторно перехоплено:", error)


# ============================================================
# 10. РОБОТА ЗІ СТУДЕНТАМИ
# ============================================================

def student_service_demo() -> None:
    """Демонструє додавання, пошук і збереження студентів."""
    print("\n6. СИСТЕМА ОБРОБКИ ДАНИХ СТУДЕНТІВ")

    service = StudentService()

    try:
        service.add_student(1, "Анна Коваленко", 95)
        service.add_student(2, "Олег Петренко", 88.5)
        service.add_student(3, "Марія Іваненко", 92)

        print("Додані студенти:")

        for student in service.students:
            print(student.describe())

        service.save_to_file("data/students.txt")
        print("Дані успішно збережено у файл.")

        print("\nРезультат пошуку:")
        print(service.find_student(2).describe())

    except AppError as error:
        print("Помилка:", error)


# ============================================================
# 11. ДЕМОНСТРАЦІЯ МОДУЛЬНОСТІ ТА ІМПОРТІВ
# ============================================================

def module_demo() -> None:
    """Демонструє основні принципи організації модулів."""
    print("\n7. МОДУЛЬНІСТЬ ТА ІМПОРТИ")

    # Імітація абсолютного імпорту:
    # import student_records.core.service

    # Імітація вибіркового імпорту:
    # from student_records import Student, StudentService

    # Приклад відносного імпорту в пакетній структурі:
    # from ..utils.handlers import log_and_reraise

    print("Приклад абсолютного імпорту:")
    print("import student_records.core.service")

    print("\nПриклад вибіркового імпорту:")
    print("from student_records import Student, StudentService")

    print("\nПриклад відносного імпорту:")
    print("from ..utils.handlers import log_and_reraise")

    print("\n__all__ визначає доступні для імпорту об'єкти.")
    print("__name__:", __name__)


# ============================================================
# 12. ДОКУМЕНТАЦІЯ ТА ІНТРОСПЕКЦІЯ
# ============================================================

def documentation_demo() -> None:
    """Демонструє засоби перегляду документації Python."""
    print("\n8. ДОКУМЕНТАЦІЯ ТА ІНТРОСПЕКЦІЯ")

    print("Документація класу Student:")
    print(Student.__doc__)

    print("\nІм'я класу:")
    print(Student.__name__)

    print("\nАнотації класу:")
    print(Student.__annotations__)

    print("\nДокументація методу validate:")
    print(Student.validate.__doc__)

    print("\nДовідкова інформація про клас:")
    help(Student)


# ============================================================
# 13. ГОЛОВНА ФУНКЦІЯ
# ============================================================

def main() -> None:
    """Запускає всі завдання лабораторної роботи."""
    print("=" * 60)
    print("ЛАБОРАТОРНА РОБОТА №3")
    print("Обробка виключень, модульність та документація Python")
    print("=" * 60)

    basic_exception_demo()
    full_exception_demo()
    multiple_exception_demo()
    custom_exception_demo()
    reraising_demo()
    student_service_demo()
    module_demo()
    documentation_demo()

    print("\nУсі завдання лабораторної роботи виконано.")


# ============================================================
# 14. ЗАПУСК ПРОГРАМИ
# ============================================================

if __name__ == "__main__":
    main()
