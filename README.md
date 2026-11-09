# Metodichka
МЕТОДИЧЕСКИЕ УКАЗАНИЯ К ПРАКТИЧЕСКОЙ РАБОТЕ
Интеграция векторного графического редактора на C# (Windows Forms, .NET 8.0)
с базой данных Microsoft SQL Server 2019 через ADO.NET
Дисциплина:	Базы данных / Разработка программных модулей (09.02.07)
Программный стек:	Microsoft SQL Server 2019, SSMS, Visual Studio 2022, C# 12.0, .NET 8.0
Слой доступа к данным:	ADO.NET (пакет Microsoft.Data.SqlClient, без использования устаревших .mdf)

1. В чём суть практической работы
В рамках первой работы вы разработали настольное приложение — графический редактор («Mini-CAD»), который хранит фигуры в оперативной памяти и локальных JSON-файлах. Суть данного сквозного задания — превратить автономную «рисовалку» в полноценную клиент-серверную информационную систему, подключив её к СУБД Microsoft SQL Server 2019.
Схема и расположенные на холсте фигуры теперь должны централизованно сохраняться в реляционной базе данных: каждая фигура раскладывается на поля нормализованной таблицы (тип, геометрия, параметры пера и заливки, текст), а пользователи получают возможность просматривать каталог проектов, загружать любые сохраненные ранее схемы обратно на холст для продолжения работы, отслеживать статистику и версии.
2. Используемое программное обеспечение
•	СУБД: Microsoft SQL Server 2019 (редакции Developer, Express или Standard). База данных развертывается как полноценный локальный сервер.
•	Среда управления БД: SQL Server Management Studio (SSMS 19 или новее). Именно в SSMS создается база данных, пишутся DDL-скрипты таблиц, ограничений целостности, триггеров и хранимых процедур.
•	Среда разработки клиента: Microsoft Visual Studio 2022 (версия 17.8+). Используется созданный ранее проект Windows Forms на платформе .NET 8.0 (LTS).
•	Библиотека доступа к БД: Microsoft.Data.SqlClient (ADO.NET). Современный высокопроизводительный драйвер для взаимодействия C# с MS SQL Server.
3. Архитектура подключения: как «цеплять» БД к проекту
Важное архитектурное правило для .NET 8.0:
В современном .NET 8.0 устаревший подход добавления локального файла базы прямо в Solution Explorer (через 'Add -> New Item -> Service-based Database' с файлом .mdf) НЕ ИСПОЛЬЗУЕТСЯ и не поддерживается!
База данных DiagramDb создается автономно на сервере MS SQL Server 2019 через SSMS. Приложение WinForms взаимодействует с ней по сети через строку подключения (Connection String) и ADO.NET.

Шаги по подготовке проекта в Visual Studio 2022:
•	Шаг 3.1: В Solution Explorer щелкните правой кнопкой мыши по проекту и выберите «Управление пакетами NuGet...» (Manage NuGet Packages).
•	Шаг 3.2: Перейдите на вкладку «Обзор» (Browse), введите в строку поиска Microsoft.Data.SqlClient и установите последнюю стабильную версию пакета.
•	Шаг 3.3: Сформируйте корректную строку подключения к локальному серверу SQL Server 2019.
// Строки подключения для файла конфигурации или сервиса БД:

// Вариант 1: Экземпляр SQL Server по умолчанию (localhost):
string connStr = "Server=localhost;Database=DiagramDb;Integrated Security=True;TrustServerCertificate=True;";

// Вариант 2: Если установлен именованный экземпляр SQLEXPRESS:
string connStr = "Server=.\SQLEXPRESS;Database=DiagramDb;Integrated Security=True;TrustServerCertificate=True;";

// Внимание: параметр TrustServerCertificate=True ОБЯЗАТЕЛЕН для Microsoft.Data.SqlClient, 
// иначе при подключении к локальному серверу без покупного SSL возникнет исключение AuthenticationException!

4. Проектирование базы данных DiagramDb (в SSMS)
Студенты открывают SQL Server Management Studio (SSMS), подключаются к своему серверу и создают базу данных. Схема нормализована до 3-й нормальной формы и исключает дублирование строковых названий типов фигур.
Таблица	Ключевые поля	Назначение и связи
Projects	ProjectID (PK, INT IDENTITY)
Title (NVARCHAR(100))
Author (NVARCHAR(50))
CreatedAt, UpdatedAt (DATETIME2)
CanvasWidth, CanvasHeight (INT)	Каталог схем. Хранит метаданные проекта и исходные размеры холста редактора.
ShapeTypes	ShapeTypeID (PK, INT IDENTITY)
TypeName (NVARCHAR(50), UNIQUE)
DefaultWidth, DefaultHeight (INT)	Справочник типов фигур (Rectangle, Ellipse, Rhombus, TextBlock). Предотвращает дублирование.
Shapes	ShapeID (PK, INT IDENTITY)
ProjectID (FK -> Projects)
ShapeTypeID (FK -> ShapeTypes)
X, Y, Width, Height (INT)
StrokeColor, FillColor (NVARCHAR(9))
StrokeWidth (FLOAT), TextContent (NVARCHAR(255))
ZOrder (INT)	Конкретные геометрические объекты холста. Связана с проектом каскадным удалением (ON DELETE CASCADE).
ProjectHistory	HistoryID (PK, INT IDENTITY)
ProjectID (FK -> Projects)
ActionType (NVARCHAR(20))
Timestamp (DATETIME2)
SnapshotCount (INT)	Журнал аудита/версионирования. Заполняется триггером при любом изменении схемы.

Скрипт создания БД, таблиц и начальных данных (T-SQL):
CREATE DATABASE DiagramDb;
GO
USE DiagramDb;
GO

-- 1. Справочник типов фигур
CREATE TABLE ShapeTypes (
    ShapeTypeID INT IDENTITY(1,1) PRIMARY KEY,
    TypeName NVARCHAR(50) NOT NULL UNIQUE,
    DefaultWidth INT NOT NULL DEFAULT 80,
    DefaultHeight INT NOT NULL DEFAULT 50
);

-- Заполнение начальными системными типами
INSERT INTO ShapeTypes (TypeName, DefaultWidth, DefaultHeight)
VALUES ('Rectangle', 80, 50), ('Ellipse', 80, 60), ('Rhombus', 90, 60), ('TextBlock', 120, 40);

-- 2. Таблица проектов
CREATE TABLE Projects (
    ProjectID INT IDENTITY(1,1) PRIMARY KEY,
    Title NVARCHAR(100) NOT NULL,
    Author NVARCHAR(50) NOT NULL DEFAULT 'Студент',
    CreatedAt DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    UpdatedAt DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    CanvasWidth INT NOT NULL CHECK (CanvasWidth > 0),
    CanvasHeight INT NOT NULL CHECK (CanvasHeight > 0)
);

-- 3. Таблица фигур холста
CREATE TABLE Shapes (
    ShapeID INT IDENTITY(1,1) PRIMARY KEY,
    ProjectID INT NOT NULL FOREIGN KEY REFERENCES Projects(ProjectID) ON DELETE CASCADE,
    ShapeTypeID INT NOT NULL FOREIGN KEY REFERENCES ShapeTypes(ShapeTypeID),
    X INT NOT NULL,
    Y INT NOT NULL,
    Width INT NOT NULL CHECK (Width >= 10),
    Height INT NOT NULL CHECK (Height >= 10),
    StrokeColor NVARCHAR(9) NOT NULL DEFAULT '#000000',
    FillColor NVARCHAR(9) NOT NULL DEFAULT '#87CEEB',
    StrokeWidth FLOAT NOT NULL DEFAULT 2.0 CHECK (StrokeWidth > 0),
    TextContent NVARCHAR(255) NULL,
    ZOrder INT NOT NULL DEFAULT 0
);

-- 4. Таблица журнала аудита
CREATE TABLE ProjectHistory (
    HistoryID INT IDENTITY(1,1) PRIMARY KEY,
    ProjectID INT NOT NULL FOREIGN KEY REFERENCES Projects(ProjectID) ON DELETE CASCADE,
    ActionType NVARCHAR(20) NOT NULL,
    Timestamp DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    SnapshotCount INT NOT NULL
);
GO

5. Серверная бизнес-логика: Процедуры, Представление и Триггер
5.1. Хранимая процедура сохранения проекта (sp_SaveProject)
Процедура выполняет вставку или обновление проекта и очищает старый состав фигур в рамках транзакции:
CREATE OR ALTER PROCEDURE sp_SaveProject
    @ProjectID INT OUTPUT,
    @Title NVARCHAR(100),
    @Author NVARCHAR(50),
    @CanvasWidth INT,
    @CanvasHeight INT
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRANSACTION;
    BEGIN TRY
        IF @ProjectID IS NULL OR @ProjectID <= 0 OR NOT EXISTS (SELECT 1 FROM Projects WHERE ProjectID = @ProjectID)
        BEGIN
            INSERT INTO Projects (Title, Author, CanvasWidth, CanvasHeight)
            VALUES (@Title, @Author, @CanvasWidth, @CanvasHeight);
            SET @ProjectID = SCOPE_IDENTITY();
        END
        ELSE
        BEGIN
            UPDATE Projects 
            SET Title = @Title, Author = @Author, CanvasWidth = @CanvasWidth, 
                CanvasHeight = @CanvasHeight, UpdatedAt = SYSDATETIME()
            WHERE ProjectID = @ProjectID;

            -- Удаляем старые фигуры перед перезаписью нового состояния
            DELETE FROM Shapes WHERE ProjectID = @ProjectID;
        END

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO

5.2. Хранимая процедура загрузки фигур проекта (sp_GetProjectShapes)
CREATE OR ALTER PROCEDURE sp_GetProjectShapes
    @ProjectID INT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT 
        s.ShapeID, s.X, s.Y, s.Width, s.Height, 
        s.StrokeColor, s.FillColor, s.StrokeWidth, s.TextContent, s.ZOrder,
        st.TypeName
    FROM Shapes s
    INNER JOIN ShapeTypes st ON s.ShapeTypeID = st.ShapeTypeID
    WHERE s.ProjectID = @ProjectID
    ORDER BY s.ZOrder ASC;
END;
GO

5.3. Аналитическое представление (v_ProjectStatistics)
CREATE OR ALTER VIEW v_ProjectStatistics
AS
SELECT 
    p.ProjectID AS [ID],
    p.Title AS [Название схемы],
    p.Author AS [Автор],
    p.UpdatedAt AS [Последнее изменение],
    COUNT(s.ShapeID) AS [Всего фигур],
    ISNULL(SUM(CASE WHEN st.TypeName = 'Rhombus' THEN 1 ELSE 0 END), 0) AS [Блоков условий],
    ISNULL(SUM(CASE WHEN st.TypeName = 'TextBlock' THEN 1 ELSE 0 END), 0) AS [Текстовых заметок]
FROM Projects p
LEFT JOIN Shapes s ON p.ProjectID = s.ProjectID
LEFT JOIN ShapeTypes st ON s.ShapeTypeID = st.ShapeTypeID
GROUP BY p.ProjectID, p.Title, p.Author, p.UpdatedAt;
GO

5.4. Триггер аудита версий (trg_Projects_Audit)
CREATE OR ALTER TRIGGER trg_Projects_Audit
ON Projects
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO ProjectHistory (ProjectID, ActionType, SnapshotCount)
    SELECT 
        i.ProjectID,
        CASE WHEN EXISTS (SELECT 1 FROM deleted) THEN 'UPDATE' ELSE 'INSERT' END,
        (SELECT COUNT(*) FROM Shapes WHERE ProjectID = i.ProjectID)
    FROM inserted i;
END;
GO

6. Клиентская разработка в Visual Studio 2022 (C# WinForms)
Главное архитектурное требование — соблюдение многоуровневой архитектуры. В классе MainForm.cs категорически запрещено писать SQL-запросы! Вся работа с базой данных инкапсулируется в сервисном классе DatabaseService.cs.
6.1. Класс сервиса базы данных (Services/DatabaseService.cs)
using System.Data;
using Microsoft.Data.SqlClient;
using MiniCad.Models;

namespace MiniCad.Services;

public class DatabaseService
{
    private readonly string _connectionString = 
        "Server=localhost;Database=DiagramDb;Integrated Security=True;TrustServerCertificate=True;";

    // Получение списка проектов через представление
    public async Task<DataTable> GetProjectsSummaryAsync()
    {
        using var conn = new SqlConnection(_connectionString);
        await conn.OpenAsync();
        using var cmd = new SqlCommand("SELECT * FROM v_ProjectStatistics ORDER BY [Последнее изменение] DESC", conn);
        using var reader = await cmd.ExecuteReaderAsync();
        var dt = new DataTable();
        dt.Load(reader);
        return dt;
    }

    // Сохранение проекта и его фигур в одной транзакции
    public async Task<int> SaveProjectAsync(int? projectId, string title, List<Shape> shapes, int width, int height)
    {
        using var conn = new SqlConnection(_connectionString);
        await conn.OpenAsync();
        using var trans = conn.BeginTransaction();

        try
        {
            // 1. Вызов процедуры сохранения проекта
            using var cmdProj = new SqlCommand("sp_SaveProject", conn, trans)
            {
                CommandType = CommandType.StoredProcedure
            };
            var idParam = new SqlParameter("@ProjectID", SqlDbType.Int)
            {
                Direction = ParameterDirection.InputOutput,
                Value = (object?)projectId ?? DBNull.Value
            };
            cmdProj.Parameters.Add(idParam);
            cmdProj.Parameters.AddWithValue("@Title", title);
            cmdProj.Parameters.AddWithValue("@Author", Environment.UserName);
            cmdProj.Parameters.AddWithValue("@CanvasWidth", width);
            cmdProj.Parameters.AddWithValue("@CanvasHeight", height);
            await cmdProj.ExecuteNonQueryAsync();

            int savedProjectId = (int)idParam.Value;

            // 2. Пакетная вставка фигур
            int order = 0;
            foreach (var s in shapes)
            {
                string typeName = s switch
                {
                    RectangleShape => "Rectangle",
                    EllipseShape => "Ellipse",
                    RhombusShape => "Rhombus",
                    _ => "Rectangle"
                };

                using var cmdShape = new SqlCommand(@"
                    INSERT INTO Shapes (ProjectID, ShapeTypeID, X, Y, Width, Height, StrokeColor, FillColor, StrokeWidth, ZOrder)
                    SELECT @Pid, ShapeTypeID, @X, @Y, @W, @H, @Stroke, @Fill, @Sw, @Z
                    FROM ShapeTypes WHERE TypeName = @TypeName;", conn, trans);

                cmdShape.Parameters.AddWithValue("@Pid", savedProjectId);
                cmdShape.Parameters.AddWithValue("@TypeName", typeName);
                cmdShape.Parameters.AddWithValue("@X", s.X);
                cmdShape.Parameters.AddWithValue("@Y", s.Y);
                cmdShape.Parameters.AddWithValue("@W", s.Width);
                cmdShape.Parameters.AddWithValue("@H", s.Height);
                cmdShape.Parameters.AddWithValue("@Stroke", ColorTranslator.ToHtml(s.StrokeColor));
                cmdShape.Parameters.AddWithValue("@Fill", ColorTranslator.ToHtml(s.FillColor));
                cmdShape.Parameters.AddWithValue("@Sw", s.StrokeWidth);
                cmdShape.Parameters.AddWithValue("@Z", order++);
                await cmdShape.ExecuteNonQueryAsync();
            }

            await trans.CommitAsync();
            return savedProjectId;
        }
        catch
        {
            await trans.RollbackAsync();
            throw;
        }
    }

    // Загрузка фигур проекта и фабричное восстановление объектов
    public async Task<List<Shape>> LoadProjectShapesAsync(int projectId)
    {
        var result = new List<Shape>();
        using var conn = new SqlConnection(_connectionString);
        await conn.OpenAsync();

        using var cmd = new SqlCommand("sp_GetProjectShapes", conn) { CommandType = CommandType.StoredProcedure };
        cmd.Parameters.AddWithValue("@ProjectID", projectId);

        using var reader = await cmd.ExecuteReaderAsync();
        while (await reader.ReadAsync())
        {
            string type = reader.GetString(reader.GetOrdinal("TypeName"));
            int x = reader.GetInt32(reader.GetOrdinal("X"));
            int y = reader.GetInt32(reader.GetOrdinal("Y"));
            int w = reader.GetInt32(reader.GetOrdinal("Width"));
            int h = reader.GetInt32(reader.GetOrdinal("Height"));
            string strokeHex = reader.GetString(reader.GetOrdinal("StrokeColor"));
            string fillHex = reader.GetString(reader.GetOrdinal("FillColor"));
            float strokeW = (float)reader.GetDouble(reader.GetOrdinal("StrokeWidth"));

            // Фабричное создание нужного типа фигуры по данным из БД
            Shape shape = type switch
            {
                "Rectangle" => new RectangleShape(),
                "Ellipse" => new EllipseShape(),
                "Rhombus" => new RhombusShape(),
                _ => new RectangleShape()
            };

            shape.X = x;
            shape.Y = y;
            shape.Width = w;
            shape.Height = h;
            shape.StrokeColor = ColorTranslator.FromHtml(strokeHex);
            shape.FillColor = ColorTranslator.FromHtml(fillHex);
            shape.StrokeWidth = strokeW;

            result.Add(shape);
        }
        return result;
    }
}

6.2. Создание формы каталога проектов (Forms/ProjectsCatalogForm.cs)
Создайте новую форму ProjectsCatalogForm с элементом DataGridView и кнопками «Открыть схему», «Удалить», «Обновить». В событии загрузки формы таблица заполняется вызовом метода GetProjectsSummaryAsync().
7. Что и как студент демонстрирует на защите
Защита лабораторной работы состоит из трех последовательных этапов:
•	1. Демонстрация в SSMS: Студент открывает SQL Server Management Studio 19, показывает структуру таблиц DiagramDb, связи (FK, PK), демонстрирует вызов представления v_ProjectStatistics и журнала ProjectHistory.
•	2. Аудит исходного кода: Преподаватель проверяет чистоту архитектуры: отсутствие SQL-запросов в MainForm.cs, параметризацию команд (защита от SQL Injection), использование конструкций using для освобождения соединений, транзакционность сохранения.
•	3. Сквозной сценарий в приложении: Студент запускает редактор, строит сложную схему из 4–5 фигур разных типов, нажимает «Сохранить в БД», вводит название. Далее холст полностью очищается (белый лист). Студент открывает форму «Каталог проектов», выбирает схему, нажимает «Загрузить» — все фигуры восстанавливаются с точными координатами и цветами. В SSMS выполняется проверочный SELECT * FROM Shapes WHERE ProjectID = ...
8. Критерии оценивания практической работы
Оценка	Требования к SQL / MS SQL Server 2019	Интеграция с WinForms (.NET 8)
«Удовлетворительно»
(3 балла)	Разработан DDL-скрипт 3 таблиц (Projects, ShapeTypes, Shapes) со всеми первичными и внешними ключами. База заполнена начальными данными.	Программа умеет сохранять схему в БД через простые SQL-запросы (INSERT). В коде нет утечек соединений (using SqlConnection).
«Хорошо»
(4 балла)	Созданы хранимые процедуры сохранения и выборки (sp_SaveProject, sp_GetProjectShapes) с транзакциями. Реализовано представление v_ProjectStatistics.	В редакторе создана отдельная форма/диалог «Каталог проектов» с DataGridView. Работает сохранение и открытие проекта из БД с полной перерисовкой холста.
«Отлично»
(5 баллов)	Реализован триггер версионирования/аудита (ProjectHistory), транзакции с обработкой ошибок (TRY...CATCH), каскадное удаление (ON DELETE CASCADE).	Все операции с БД вынесены в асинхронный сервис (async/await, Task). Реализован поиск/фильтрация проектов по автору и дате в DataGridView.

