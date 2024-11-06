CREATE DATABASE TestDB;
USE TestDB;
 
CREATE TABLE Users (
    UserID INT PRIMARY KEY,
    Username VARCHAR(50),
    Balance DECIMAL(10, 2)
);
 

INSERT INTO Users (UserID, Username, Balance) VALUES (1, 'Alice', 100.00);
INSERT INTO Users (UserID, Username, Balance) VALUES (2, 'Bob', 200.00);

START TRANSACTION;
 
DECLARE @UserInput NVARCHAR(50);
SET @UserInput = '1; DROP TABLE Users; --'; 
 
DECLARE @SQL NVARCHAR(MAX);
SET @SQL = N'SELECT * FROM Users WHERE UserID = ' + @UserInput;
 
EXEC sp_executesql @SQL; 

UPDATE Users
SET Balance = Balance - 50.00
WHERE UserID = 1;

WAITFOR DELAY '00:00:05';

COMMIT TRANSACTION;

START TRANSACTION;

UPDATE Users
SET Balance = Balance + 30.00
WHERE UserID = 1;

WAITFOR DELAY '00:00:05';

COMMIT TRANSACTION;

SELECT * FROM Users;

-- DROP TABLE Users;
-- DROP DATABASE TestDB;
