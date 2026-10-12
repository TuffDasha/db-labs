```powershell
docker run --name lab3-postgres -e POSTGRES_PASSWORD=postgres -p 5433:5432 -d postgres:16
docker ps
```

```powershell
docker exec -it lab3-postgres psql -U postgres
```

```sql
CREATE DATABASE library_lab3;
\c library_lab3
```

```sql
CREATE TABLE authors (
    author_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    birth_year INTEGER CHECK (birth_year >= 1000 AND birth_year <= EXTRACT(YEAR FROM CURRENT_DATE)),
    country VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

```sql
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    isbn VARCHAR(17) UNIQUE NOT NULL,
    title VARCHAR(200) NOT NULL,
    author_id INTEGER NOT NULL,
    publication_year INTEGER CHECK (publication_year >= 1450),
    genre VARCHAR(50),
    total_copies INTEGER DEFAULT 1 CHECK (total_copies >= 0),
    available_copies INTEGER DEFAULT 1 CHECK (available_copies >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (author_id) REFERENCES authors(author_id) ON DELETE RESTRICT,
    CHECK (available_copies <= total_copies)
);
```

```sql
CREATE TABLE readers (
    reader_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(20),
    library_card_number VARCHAR(20) UNIQUE NOT NULL,
    registration_date DATE DEFAULT CURRENT_DATE,
    status VARCHAR(20) DEFAULT 'Active' CHECK (status IN ('Active', 'Blocked', 'Inactive'))
);
```

```sql
CREATE TABLE lendings (
    lending_id SERIAL PRIMARY KEY,
    book_id INTEGER NOT NULL,
    reader_id INTEGER NOT NULL,
    lending_date DATE DEFAULT CURRENT_DATE,
    due_date DATE NOT NULL,
    return_date DATE,
    FOREIGN KEY (book_id) REFERENCES books(book_id) ON DELETE RESTRICT,
    FOREIGN KEY (reader_id) REFERENCES readers(reader_id) ON DELETE RESTRICT,
    CHECK (due_date > lending_date),
    CHECK (return_date IS NULL OR return_date >= lending_date)
);
```

```sql
\dt
\d books
```

```sql
INSERT INTO authors (first_name, last_name, birth_year, country)
VALUES
    ('Тарас', 'Шевченко', 1814, 'Україна'),
    ('Іван', 'Франко', 1856, 'Україна'),
    ('Леся', 'Українка', 1871, 'Україна'),
    ('Михайло', 'Коцюбинський', 1864, 'Україна'),
    ('Панас', 'Мирний', 1849, 'Україна');
```

```sql
INSERT INTO books (isbn, title, author_id, publication_year, genre, total_copies, available_copies)
VALUES
    ('978-966-03-4561-2', 'Кобзар', 1, 1840, 'Поезія', 10, 8),
    ('978-966-03-4562-9', 'Захар Беркут', 2, 1883, 'Історична проза', 5, 4),
    ('978-966-03-4563-6', 'Лісова пісня', 3, 1911, 'Драма', 7, 7),
    ('978-966-03-4564-3', 'Тіні забутих предків', 4, 1911, 'Повість', 6, 5),
    ('978-966-03-4565-0', 'Хіба ревуть воли, як ясла повні', 5, 1880, 'Повість', 4, 4);
```

```sql
INSERT INTO readers (first_name, last_name, email, phone, library_card_number)
VALUES
    ('Олена', 'Петренко', 'olena.petrenko@email.com', '+380501234567', 'LIB-2024-001'),
    ('Андрій', 'Коваленко', 'andrii.kovalenko@email.com', '+380502345678', 'LIB-2024-002'),
    ('Марія', 'Сидоренко', 'maria.sydorenko@email.com', '+380503456789', 'LIB-2024-003'),
    ('Іван', 'Мельник', 'ivan.melnyk@email.com', '+380504567890', 'LIB-2024-004');
```

```sql
SELECT * FROM authors;
SELECT book_id, isbn, title, author_id, total_copies, available_copies FROM books;
SELECT reader_id, first_name, last_name, library_card_number, status FROM readers;
```

```sql
SELECT book_id, title, available_copies FROM books WHERE book_id = 1;
```

```sql
UPDATE books
SET available_copies = available_copies - 1
WHERE book_id = 1 AND available_copies > 0;
```

```sql
SELECT book_id, title, available_copies FROM books WHERE book_id = 1;
```

```sql
BEGIN;

INSERT INTO lendings (book_id, reader_id, due_date)
VALUES (1, 1, CURRENT_DATE + INTERVAL '14 days');

UPDATE books
SET available_copies = available_copies - 1
WHERE book_id = 1 AND available_copies > 0;

SELECT * FROM lendings WHERE lending_id = LASTVAL();
SELECT available_copies FROM books WHERE book_id = 1;

COMMIT;
```

```sql
BEGIN;
DELETE FROM authors WHERE author_id = 1;
```

```sql
ROLLBACK;
```

```sql
SELECT * FROM authors WHERE author_id = 1;
```

```sql
INSERT INTO readers (first_name, last_name, email, phone, library_card_number, status)
VALUES ('Тест', 'Неактивний', 'test.inactive@email.com', '+380505678901', 'LIB-2024-005', 'Inactive');

SELECT * FROM readers WHERE status = 'Inactive' AND reader_id NOT IN (
    SELECT DISTINCT reader_id FROM lendings WHERE return_date IS NULL
);

DELETE FROM readers
WHERE status = 'Inactive' AND reader_id NOT IN (
    SELECT DISTINCT reader_id FROM lendings WHERE return_date IS NULL
);

SELECT reader_id, first_name, last_name, status FROM readers;
```

```sql
CREATE TABLE reservations (
    reservation_id SERIAL PRIMARY KEY,
    book_id INTEGER NOT NULL,
    reader_id INTEGER NOT NULL,
    reservation_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expiration_date TIMESTAMP NOT NULL,
    status VARCHAR(20) DEFAULT 'Active' CHECK (status IN ('Active', 'Fulfilled', 'Cancelled', 'Expired')),
    FOREIGN KEY (book_id) REFERENCES books(book_id) ON DELETE CASCADE,
    FOREIGN KEY (reader_id) REFERENCES readers(reader_id) ON DELETE CASCADE
);

CREATE TABLE audit_log (
    log_id SERIAL PRIMARY KEY,
    table_name VARCHAR(50) NOT NULL,
    operation VARCHAR(10) NOT NULL,
    record_id INTEGER NOT NULL,
    old_data JSONB,
    new_data JSONB,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    changed_by VARCHAR(100) DEFAULT CURRENT_USER
);
```

```sql
\dt
\d reservations
```

```sql
INSERT INTO reservations (book_id, reader_id, expiration_date)
VALUES (1, 2, CURRENT_TIMESTAMP + INTERVAL '7 days');

SELECT lending_id, return_date FROM lendings WHERE lending_id = 1;
SELECT book_id, available_copies FROM books WHERE book_id = 1;
SELECT reservation_id, status FROM reservations;
```

```sql
BEGIN;

UPDATE lendings
SET return_date = CURRENT_DATE
WHERE lending_id = 1 AND return_date IS NULL;

UPDATE books
SET available_copies = available_copies + 1
WHERE book_id = (SELECT book_id FROM lendings WHERE lending_id = 1);

SELECT reservation_id, reader_id
FROM reservations
WHERE book_id = (SELECT book_id FROM lendings WHERE lending_id = 1)
    AND status = 'Active'
ORDER BY reservation_date
LIMIT 1;

UPDATE reservations
SET status = 'Fulfilled'
WHERE reservation_id = (
    SELECT reservation_id
    FROM reservations
    WHERE book_id = (SELECT book_id FROM lendings WHERE lending_id = 1)
        AND status = 'Active'
    ORDER BY reservation_date
    LIMIT 1
);

COMMIT;
```

```sql
BEGIN;

INSERT INTO authors (first_name, last_name, birth_year, country)
VALUES ('Ольга', 'Кобилянська', 1863, 'Україна');

SAVEPOINT after_author;

INSERT INTO books (isbn, title, author_id, publication_year, genre, total_copies, available_copies)
VALUES ('978-966-03-4566-7', 'Земля', (SELECT author_id FROM authors WHERE last_name = 'Кобилянська'), 1902, 'Роман', 3, 3);

SAVEPOINT after_book;

INSERT INTO books (isbn, title, author_id, publication_year, genre)
VALUES ('978-966-03-4566-7', 'Інша книга', (SELECT author_id FROM authors WHERE last_name = 'Кобилянська'), 1905, 'Роман');

ROLLBACK TO SAVEPOINT after_author;

INSERT INTO books (isbn, title, author_id, publication_year, genre, total_copies, available_copies)
VALUES ('978-966-03-4567-4', 'Земля', (SELECT author_id FROM authors WHERE last_name = 'Кобилянська'), 1902, 'Роман', 3, 3);

COMMIT;
```

```sql
BEGIN;

INSERT INTO authors (first_name, last_name, birth_year, country)
VALUES ('Тестовий', 'Автор', 1900, 'Україна');

INSERT INTO books (isbn, title, author_id, publication_year, genre, total_copies, available_copies)
VALUES ('978-966-00-0000-0', 'Тестова книга', (SELECT author_id FROM authors WHERE last_name = 'Автор'), 2000, 'Тест', 1, 0);

INSERT INTO reservations (book_id, reader_id, expiration_date)
VALUES ((SELECT book_id FROM books WHERE isbn = '978-966-00-0000-0'), 1, CURRENT_TIMESTAMP + INTERVAL '7 days');

COMMIT;

SELECT b.book_id, b.title, r.reservation_id
FROM books b
LEFT JOIN reservations r ON b.book_id = r.book_id
WHERE b.isbn = '978-966-00-0000-0';

DELETE FROM books WHERE isbn = '978-966-00-0000-0';

SELECT * FROM reservations WHERE book_id NOT IN (SELECT book_id FROM books);

SELECT reservation_id, book_id, reader_id, status FROM reservations;
```

```sql
CREATE OR REPLACE FUNCTION log_books_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, record_id, new_data)
        VALUES ('books', 'INSERT', NEW.book_id, row_to_json(NEW)::jsonb);
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, record_id, old_data, new_data)
        VALUES ('books', 'UPDATE', NEW.book_id, row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb);
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, record_id, old_data)
        VALUES ('books', 'DELETE', OLD.book_id, row_to_json(OLD)::jsonb);
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER books_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON books
FOR EACH ROW
EXECUTE FUNCTION log_books_changes();
```

```sql
UPDATE books SET available_copies = available_copies + 1 WHERE book_id = 1;

SELECT log_id, operation, record_id,
      new_data->>'title' as title,
      old_data->>'available_copies' as old_copies,
      new_data->>'available_copies' as new_copies,
      changed_at
FROM audit_log
WHERE table_name = 'books'
ORDER BY changed_at DESC;
```

```powershell
docker exec -it lab3-postgres psql -U postgres -d library_lab3
```

```sql
BEGIN;
SELECT * FROM books WHERE book_id = 1 FOR UPDATE;
```

```sql
BEGIN;
SELECT * FROM books WHERE book_id = 1 FOR UPDATE;
```

```sql
UPDATE books SET available_copies = available_copies - 1 WHERE book_id = 1;
COMMIT;
```

```sql
UPDATE books SET available_copies = available_copies - 1 WHERE book_id = 1;
COMMIT;
```

```sql
SELECT book_id, title, available_copies FROM books WHERE book_id = 1;
```

```sql
CREATE OR REPLACE FUNCTION log_books_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, record_id, new_data)
        VALUES ('books', 'INSERT', NEW.book_id, row_to_json(NEW)::jsonb);
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, record_id, old_data, new_data)
        VALUES ('books', 'UPDATE', NEW.book_id, row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb);
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, record_id, old_data)
        VALUES ('books', 'DELETE', OLD.book_id, row_to_json(OLD)::jsonb);
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```
