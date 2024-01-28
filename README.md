## SQL ile Yaş Doğrulama Fonksiyonu Yazma

``` sql
CREATE FUNCTION CheckAge (@dateOfBirth DATE, @age INT)
RETURNS NVARCHAR(100)
AS
BEGIN
    DECLARE @yearPass INT;
    DECLARE @monthPass INT;
    DECLARE @dayPass INT;
    DECLARE @result NVARCHAR(100);

    SELECT 
        @yearPass = DATEDIFF(YEAR, @dateOfBirth, GETDATE()),
        @monthPass = DATEDIFF(MONTH, @dateOfBirth, DATEADD(YEAR, @yearPass, @dateOfBirth)),
        @dayPass = DATEDIFF(DAY, DATEADD(MONTH, @monthPass, DATEADD(YEAR, @yearPass, @dateOfBirth)), GETDATE());

    IF @yearPass >= @age
    BEGIN
        IF @monthPass >= @age * 12
        BEGIN
            IF @dayPass >= @age * 365
            BEGIN
                SET @result = N'Yıl, ay ve gün olarak doldurmuştur.';
            END
            ELSE
            BEGIN
                SET @result = N'Yıl ve ay olarak doldurmuştur, gün olarak doldurmuştur.';
            END
        END
        ELSE
        BEGIN
            SET @result = N'Yıl olarak doldurmuştur, ay ve gün olarak doldurmuştur.';
        END
    END
    ELSE
    BEGIN
        SET @result = N'Kişi henüz yıl, ay ve gün olarak yaşını doldurmamıştır.';
    END


    RETURN @result; 
END;


SELECT dbo.CheckAge('1990-01-01', 35) AS Result;

SELECT dbo.CheckAge('1985-06-15', 37) AS Result;

```

## Stored Procedure ile Ürün ve Kategori Ekleme

``` sql
CREATE PROCEDURE AddProductAndCategory
    @CategoryName NVARCHAR(50),
    @Description NVARCHAR(255),
    @ProductName NVARCHAR(100),
    @UnitPrice DECIMAL(10, 2),
    @UnitsInStock INT
AS
BEGIN
    DECLARE @CategoryID INT;

    SELECT @CategoryID = CategoryID
    FROM Categories
    WHERE CategoryName = @CategoryName;

    IF @CategoryID IS NULL
    BEGIN
        INSERT INTO Categories (CategoryName, Description)
        VALUES (@CategoryName, @Description);

        SET @CategoryID = SCOPE_IDENTITY();
    END

    INSERT INTO Products (ProductName, UnitPrice, UnitsInStock, CategoryID)
    VALUES (@ProductName, @UnitPrice, @UnitsInStock, @CategoryID);
END;

EXEC AddProductAndCategory 'Elektronik', 'Elektronik ürünler', 'Akıllı Telefon', 1500.00, 50;

EXEC AddProductAndCategory 'Ev Aletleri', 'Ev aletleri ürünleri', 'Çamaşır Makinesi', 2000.00, 30;

SELECT * FROM Products; SELECT * FROM Categories

```

## Kategori Kontrolü ve Ekleme İşlemleri için Stored Procedure Yazma

``` sql
CREATE PROCEDURE CategoryCheckAndAdd
    @categoryName NVARCHAR(15),
	@description NTEXT
AS
BEGIN
    IF NOT EXISTS (SELECT 1 FROM Categories WHERE CategoryName = @categoryName)
    BEGIN
        INSERT INTO Categories (CategoryName, Description)
        VALUES (@categoryName, @description);

        PRINT 'Category added.';
    END
    ELSE
    BEGIN
        PRINT 'This ' + @categoryName + ' is already exist in Categories tables.';
    END
END;


EXEC CategoryCheckAndAdd 'Bilgisayar','Bilgisayar donanımı';

SELECT * FROM Categories
```

## Kademeli Yerleşim Yerleri Ekleme İşlemleri için Stored Procedure Yazma

``` sql
CREATE DATABASE LocationManagement;

USE LocationManagement;

CREATE TABLE Countries (
    ID INT IDENTITY(1,1) NOT NULL,
    Name NVARCHAR(255) NOT NULL,
    CONSTRAINT PK_Countries PRIMARY KEY (ID)
);

CREATE TABLE Cities (
    ID INT IDENTITY(1,1) NOT NULL,
    Name NVARCHAR(255) NOT NULL,
    CountryID INT NOT NULL,
    CONSTRAINT PK_Cities PRIMARY KEY (ID),
    CONSTRAINT FK_Cities_Countries FOREIGN KEY (CountryID) REFERENCES Countries (ID)
);

CREATE TABLE Districts (
    ID INT IDENTITY(1,1) NOT NULL,
    Name NVARCHAR(255) NOT NULL,
    CityID INT NOT NULL,
    CONSTRAINT PK_Districts PRIMARY KEY (ID),
    CONSTRAINT FK_Districts_Cities FOREIGN KEY (CityID) REFERENCES Cities (ID)
);

CREATE TABLE Towns (
    ID INT IDENTITY(1,1) NOT NULL,
    Name NVARCHAR(255) NOT NULL,
    DistrictID INT NOT NULL,
    CONSTRAINT PK_Towns PRIMARY KEY (ID),
    CONSTRAINT FK_Towns_Districts FOREIGN KEY (DistrictID) REFERENCES Districts (ID)
);


CREATE PROCEDURE InsertLocation
(
    @CountryName NVARCHAR(255),
    @CityName NVARCHAR(255),
    @DistrictName NVARCHAR(255),
    @TownName NVARCHAR(255)
)
AS
BEGIN
    IF NOT EXISTS (
        SELECT *
        FROM Countries
        WHERE Name = @CountryName
    )
    BEGIN
        INSERT INTO Countries (Name)
        VALUES (@CountryName);
        PRINT 'Country: ' + @CountryName + ' added.';
    END
    ELSE
    BEGIN
        PRINT 'Country: ' + @CountryName + ' is already exist.';
    END

    DECLARE @CountryID INT;
    SET @CountryID = (
        SELECT ID
        FROM Countries
        WHERE Name = @CountryName
    );

    IF NOT EXISTS (
        SELECT *
        FROM Cities
        WHERE Name = @CityName
        AND CountryID = @CountryID
    )
    BEGIN
        INSERT INTO Cities (Name, CountryID)
        VALUES (@CityName, @CountryID);
        PRINT 'City: ' + @CityName + ' added.';
    END
    ELSE
    BEGIN
        PRINT  @CityName + ' is already exist in ' + @CountryName + '.';
    END

    DECLARE @CityID INT;
    SET @CityID = (
        SELECT ID
        FROM Cities
        WHERE Name = @CityName
        AND CountryID = @CountryID
    );

    IF NOT EXISTS (
        SELECT *
        FROM Districts
        WHERE Name = @DistrictName
        AND CityID = @CityID
    )
    BEGIN
        INSERT INTO Districts (Name, CityID)
        VALUES (@DistrictName, @CityID);
        PRINT 'District:' + @DistrictName + ' added.';
    END
    ELSE
    BEGIN
        PRINT @DistrictName + ' is already exist in ' + @CityName + ' city.';
    END

    DECLARE @DistrictID INT;
    SET @DistrictID = (
        SELECT ID
        FROM Districts
        WHERE Name = @DistrictName
        AND CityID = @CityID
    );

    IF NOT EXISTS (
        SELECT *
        FROM Towns
        WHERE Name = @TownName
        AND DistrictID = @DistrictID
    )
    BEGIN
        INSERT INTO Towns (Name, DistrictID)
        VALUES (@TownName, @DistrictID);
        PRINT 'Town: ' + @TownName + ' added.';
    END
    ELSE
    BEGIN
        PRINT @TownName + ' is already exist in ' + @DistrictName + ' district.';
    END
END;

EXEC InsertLocation 'Türkiye', 'İstanbul', 'Beşiktaş', 'Ortaköy';
EXEC InsertLocation 'Türkiye', 'İstanbul', 'Beşiktaş', 'Ortaköy';
```
