# Notification System Design

# Stage 1

## REST API Design

### Objective

Design a REST API for a notification system that allows students to receive, view, update, and manage notifications. The APIs use REST principles and exchange data using JSON.

---

## 1. Get All Notifications

**Method:** GET

**Endpoint:**

```
/api/notifications
```

### Headers

```
Authorization: Bearer <token>
Content-Type: application/json
```

### Response

```json
{
  "success": true,
  "notifications": [
    {
      "id": 1,
      "title": "Placement Drive",
      "message": "Affordmed placement drive starts today.",
      "notificationType": "Placement",
      "priority": "High",
      "isRead": false,
      "createdAt": "2026-06-25T10:30:00Z"
    }
  ]
}
```

---

## 2. Get Notification By ID

**Method:** GET

**Endpoint**

```
/api/notifications/{id}
```

### Headers

```
Authorization: Bearer <token>
Content-Type: application/json
```

### Response

```json
{
  "id": 1,
  "title": "Placement Drive",
  "message": "Affordmed placement drive starts today.",
  "notificationType": "Placement",
  "priority": "High",
  "isRead": false,
  "createdAt": "2026-06-25T10:30:00Z"
}
```

---

## 3. Mark Notification As Read

**Method:** PUT

**Endpoint**

```
/api/notifications/{id}/read
```

### Headers

```
Authorization: Bearer <token>
Content-Type: application/json
```

### Response

```json
{
  "success": true,
  "message": "Notification marked as read successfully."
}
```

---

## 4. Mark All Notifications As Read

**Method:** PUT

**Endpoint**

```
/api/notifications/read-all
```

### Headers

```
Authorization: Bearer <token>
Content-Type: application/json
```

### Response

```json
{
  "success": true,
  "message": "All notifications marked as read successfully."
}
```

---

## 5. Delete Notification

**Method:** DELETE

**Endpoint**

```
/api/notifications/{id}
```

### Headers

```
Authorization: Bearer <token>
Content-Type: application/json
```

### Response

```json
{
  "success": true,
  "message": "Notification deleted successfully."
}
```

---

## Notification JSON Schema

| Field | Type | Description |
|-------|------|-------------|
| id | Integer | Unique notification ID |
| title | String | Notification title |
| message | String | Notification message |
| notificationType | String | Placement, Result, Event |
| priority | String | High, Medium, Low |
| isRead | Boolean | Read status |
| createdAt | Timestamp | Notification creation time |

---

## Real-Time Notification Mechanism

The notification system uses **WebSockets** for real-time communication.

### Workflow

1. The student logs into the application.
2. The client establishes a WebSocket connection with the server.
3. Whenever a new notification is created, the server immediately pushes it to connected users.
4. The notification appears instantly without requiring the user to refresh the page.

---

# Stage 2

## Database Design

### Database Choice

I would use **PostgreSQL** because it provides:

- ACID transaction support
- Excellent indexing
- High reliability
- Better performance for structured notification data
- Easy scalability for millions of records

---

## Database Schema

### Table: notifications

| Column | Data Type | Description |
|---------|-----------|-------------|
| id | UUID | Primary Key |
| student_id | INTEGER | Student ID |
| title | VARCHAR(255) | Notification title |
| message | TEXT | Notification message |
| notification_type | VARCHAR(50) | Placement, Result, Event |
| priority | VARCHAR(20) | High, Medium, Low |
| is_read | BOOLEAN | Read status |
| created_at | TIMESTAMP | Notification creation time |

---

## Challenges As Data Grows

- Query execution becomes slower with millions of notifications.
- Database storage increases significantly.
- Response time increases under heavy traffic.
- Sorting notifications becomes expensive.

---

## Solutions

- Create indexes on **student_id**, **is_read**, and **created_at**.
- Use pagination to fetch notifications in smaller batches.
- Archive old notifications periodically.
- Cache frequently accessed notifications using Redis.

---

## SQL Queries

### Get All Notifications

```sql
SELECT *
FROM notifications
WHERE student_id = ?;
```

---

### Get Unread Notifications

```sql
SELECT *
FROM notifications
WHERE student_id = ?
AND is_read = false;
```

---

### Mark Notification As Read

```sql
UPDATE notifications
SET is_read = true
WHERE id = ?;
```

---

### Delete Notification

```sql
DELETE FROM notifications
WHERE id = ?;
```


# Stage 3

## Query Optimization

### Given Query

```sql
SELECT *
FROM notifications
WHERE student_id = 1042
AND is_read = false
ORDER BY created_at DESC;
```

---

## Why is this query slow?

When the database contains millions of notification records, the database has to scan a large number of rows before finding the required notifications. It also needs to sort the matching records by `created_at`, which increases execution time. As the number of notifications grows, the response time becomes slower and database load increases.

---

## Optimized Solution

Create a composite index on the columns used in the query.

```sql
CREATE INDEX idx_notifications_student_read_created
ON notifications(student_id, is_read, created_at DESC);
```

---

## Why this index helps

- `student_id` helps quickly find notifications for a particular student.
- `is_read` filters unread notifications efficiently.
- `created_at DESC` allows the database to return the newest notifications first without performing an additional sort.

---

## Benefits

- Faster query execution.
- Reduced database load.
- Improved response time.
- Better performance even with millions of notification records.

---

## Additional Improvements

- Use pagination to return notifications in smaller batches instead of loading all records.
- Archive old notifications that are no longer needed.
- Cache frequently accessed notifications using Redis.


# Stage 4

## Reducing Database Load

### Problem

Whenever a user opens the notifications page, the frontend requests all notifications from the backend. As the number of users increases, this results in a large number of database queries, causing high database load and slower response times.

---

## Solution

To reduce database load while ensuring users receive the latest notifications, I would implement the following techniques:

### 1. Pagination

Instead of returning all notifications at once, return a limited number of notifications per request (for example, 20 notifications).

Example:

```http
GET /api/notifications?page=1&size=20
```

This reduces the amount of data fetched from the database.

---

### 2. Caching

Store frequently accessed notifications in Redis.

- First request retrieves data from the database.
- Subsequent requests are served from the cache.
- Cache is updated whenever a new notification is created.

This significantly reduces database queries.

---

### 3. Real-Time Updates

Use WebSockets to push new notifications to users instantly instead of repeatedly polling the server.

Benefits:
- Instant notification delivery.
- Fewer unnecessary API requests.
- Lower database load.

---

### 4. Database Indexing

Create indexes on frequently searched columns.

```sql
CREATE INDEX idx_notifications
ON notifications(student_id, is_read, created_at DESC);
```

Indexes reduce query execution time.

---

## Benefits

- Lower database load.
- Faster response time.
- Better user experience.
- Improved scalability for thousands of concurrent users.


# Stage 5

## Scalable and Reliable Notification Delivery

### Problem

The system needs to send notifications to 50,000 students. If the notification service crashes or there is a network failure, some students may not receive their notifications.

---

## Solution

To build a reliable and scalable notification system, I would use a Message Queue such as RabbitMQ or Apache Kafka.

### Architecture

Notification Service → Message Queue → Notification Worker → Student

---

## Workflow

1. A new notification is created.
2. The Notification Service sends the notification to the Message Queue.
3. The queue stores the notification safely until it is processed.
4. Notification Workers read messages from the queue.
5. Workers deliver notifications to students.
6. After successful delivery, the message is removed from the queue.
7. If delivery fails, the message remains in the queue and is retried.

---

## Advantages

- Reliable message delivery.
- Notifications are not lost if the server crashes.
- Supports processing for thousands of users simultaneously.
- Easy to scale by adding more notification workers.
- Better fault tolerance and high availability.

---

## Technologies

- RabbitMQ or Apache Kafka for message queues.
- WebSockets for real-time notification delivery.
- Redis for caching.
- PostgreSQL for storing notification data.

---

## Benefits

- Handles notifications for 50,000+ students.
- Prevents notification loss during failures.
- Improves scalability.
- Reduces system downtime.
- Provides fast and reliable notification delivery.