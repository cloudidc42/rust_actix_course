# Part 085: Project: Task Management API ✅

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้าง Task Management API ที่สมบูรณ์
- ออกแบบ Projects, Tasks, Subtasks models
- ทำ Assignment และ Collaborators
- จัดการ Due Dates และ Reminders
- ทำ Task Dependencies
- รองรับ Kanban Board
- ทำ Time Tracking
- จัดการ Comments on Tasks และ File Attachments

---

## 1. โครงสร้างโปรเจกต์

```
task_api/
├── Cargo.toml
├── .env
└── src/
    ├── main.rs
    ├── errors.rs
    ├── models/
    │   ├── mod.rs
    │   ├── project.rs
    │   ├── task.rs
    │   ├── comment.rs
    │   └── time_entry.rs
    ├── handlers/
    │   ├── mod.rs
    │   ├── projects.rs
    │   ├── tasks.rs
    │   ├── subtasks.rs
    │   ├── comments.rs
    │   ├── time_tracking.rs
    │   └── kanban.rs
    └── services/
        ├── mod.rs
        ├── notification.rs
        └── dependency.rs
```

---

## 2. Cargo.toml

```toml
[package]
name = "task_api"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-cors = "0.7"
actix-multipart = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
jsonwebtoken = "9"
bcrypt = "0.15"
dotenv = "0.15"
env_logger = "0.11"
log = "0.4"
validator = { version = "0.18", features = ["derive"] }
thiserror = "1"
futures-util = "0.3"
tokio-cron-scheduler = "0.10"
```

---

## 3. Models

### 3.1 Project Model (`src/models/project.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "project_status", rename_all = "lowercase")]
pub enum ProjectStatus {
    Active,
    Completed,
    Archived,
    OnHold,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Project {
    pub id: Uuid,
    pub name: String,
    pub description: Option<String>,
    pub color: String,
    pub owner_id: Uuid,
    pub status: ProjectStatus,
    pub due_date: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct ProjectMember {
    pub project_id: Uuid,
    pub user_id: Uuid,
    pub role: MemberRole,
    pub joined_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "member_role", rename_all = "lowercase")]
pub enum MemberRole {
    Owner,
    Admin,
    Member,
    Viewer,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateProjectRequest {
    #[validate(length(min = 1, max = 100))]
    pub name: String,
    pub description: Option<String>,
    pub color: Option<String>,
    pub due_date: Option<DateTime<Utc>>,
}

#[derive(Debug, Serialize)]
pub struct ProjectWithStats {
    pub project: Project,
    pub total_tasks: i64,
    pub completed_tasks: i64,
    pub overdue_tasks: i64,
    pub members: Vec<ProjectMember>,
    pub completion_percentage: f64,
}
```

### 3.2 Task Model (`src/models/task.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq, Eq, Hash)]
#[sqlx(type_name = "task_status", rename_all = "snake_case")]
pub enum TaskStatus {
    Backlog,
    Todo,
    InProgress,
    InReview,
    Done,
    Cancelled,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "task_priority", rename_all = "lowercase")]
pub enum TaskPriority {
    Low,
    Medium,
    High,
    Urgent,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Task {
    pub id: Uuid,
    pub title: String,
    pub description: Option<String>,
    pub project_id: Uuid,
    pub parent_task_id: Option<Uuid>,
    pub assignee_id: Option<Uuid>,
    pub creator_id: Uuid,
    pub status: TaskStatus,
    pub priority: TaskPriority,
    pub labels: Vec<String>,
    pub due_date: Option<DateTime<Utc>>,
    pub start_date: Option<DateTime<Utc>>,
    pub estimated_hours: Option<f64>,
    pub position: i32,
    pub is_blocked: bool,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize)]
pub struct TaskWithDetails {
    pub task: Task,
    pub assignee: Option<UserSummary>,
    pub creator: UserSummary,
    pub subtasks: Vec<Task>,
    pub comments_count: i64,
    pub time_logged_hours: f64,
    pub attachments_count: i64,
    pub dependencies: Vec<TaskDependency>,
    pub blocked_by: Vec<TaskSummary>,
}

#[derive(Debug, Serialize)]
pub struct UserSummary {
    pub id: Uuid,
    pub username: String,
    pub avatar_url: Option<String>,
}

#[derive(Debug, Serialize, Deserialize, FromRow)]
pub struct TaskDependency {
    pub task_id: Uuid,
    pub depends_on_id: Uuid,
    pub dependency_type: String,
}

#[derive(Debug, Serialize)]
pub struct TaskSummary {
    pub id: Uuid,
    pub title: String,
    pub status: TaskStatus,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateTaskRequest {
    #[validate(length(min = 1, max = 200))]
    pub title: String,
    pub description: Option<String>,
    pub project_id: Uuid,
    pub assignee_id: Option<Uuid>,
    pub status: Option<TaskStatus>,
    pub priority: Option<TaskPriority>,
    pub labels: Option<Vec<String>>,
    pub due_date: Option<DateTime<Utc>>,
    pub start_date: Option<DateTime<Utc>>,
    pub estimated_hours: Option<f64>,
    pub parent_task_id: Option<Uuid>,
    pub depends_on: Option<Vec<Uuid>>,
}

#[derive(Debug, Deserialize)]
pub struct UpdateTaskRequest {
    pub title: Option<String>,
    pub description: Option<String>,
    pub assignee_id: Option<Uuid>,
    pub status: Option<TaskStatus>,
    pub priority: Option<TaskPriority>,
    pub labels: Option<Vec<String>>,
    pub due_date: Option<DateTime<Utc>>,
    pub estimated_hours: Option<f64>,
}

#[derive(Debug, Deserialize)]
pub struct MoveTaskRequest {
    pub status: TaskStatus,
    pub position: Option<i32>,
}

#[derive(Debug, Deserialize)]
pub struct TaskQuery {
    pub page: Option<u32>,
    pub per_page: Option<u32>,
    pub project_id: Option<Uuid>,
    pub assignee_id: Option<Uuid>,
    pub status: Option<TaskStatus>,
    pub priority: Option<TaskPriority>,
    pub search: Option<String>,
    pub due_before: Option<DateTime<Utc>>,
    pub label: Option<String>,
    pub is_overdue: Option<bool>,
}
```

### 3.3 Time Entry Model (`src/models/time_entry.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct TimeEntry {
    pub id: Uuid,
    pub task_id: Uuid,
    pub user_id: Uuid,
    pub description: Option<String>,
    pub hours: f64,
    pub logged_date: chrono::NaiveDate,
    pub started_at: Option<DateTime<Utc>>,
    pub ended_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct LogTimeRequest {
    pub hours: f64,
    pub description: Option<String>,
    pub logged_date: Option<chrono::NaiveDate>,
}

#[derive(Debug, Deserialize)]
pub struct StartTimerRequest {
    pub description: Option<String>,
}
```

---

## 4. Task Handler (`src/handlers/tasks.rs`)

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use sqlx::PgPool;
use uuid::Uuid;
use validator::Validate;

use crate::errors::AppError;
use crate::middleware::auth::require_auth;
use crate::models::task::{CreateTaskRequest, MoveTaskRequest, TaskQuery, TaskStatus, UpdateTaskRequest};

pub async fn list_tasks(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    query: web::Query<TaskQuery>,
) -> Result<HttpResponse, AppError> {
    let claims = require_auth(&req)?;
    let page = query.page.unwrap_or(1);
    let per_page = query.per_page.unwrap_or(50).min(200);
    let offset = ((page - 1) * per_page) as i64;

    let tasks = sqlx::query!(
        r#"
        SELECT t.id, t.title, t.description, t.project_id, t.assignee_id,
               t.status::text, t.priority::text, t.labels, t.due_date,
               t.estimated_hours, t.position, t.is_blocked,
               t.completed_at, t.created_at, t.updated_at,
               u.username as assignee_username,
               u.avatar_url as assignee_avatar
        FROM tasks t
        LEFT JOIN users u ON t.assignee_id = u.id
        JOIN project_members pm ON t.project_id = pm.project_id AND pm.user_id = $1
        WHERE t.parent_task_id IS NULL
          AND ($2::uuid IS NULL OR t.project_id = $2)
          AND ($3::uuid IS NULL OR t.assignee_id = $3)
          AND ($4::text IS NULL OR t.status::text = $4)
          AND ($5::text IS NULL OR t.priority::text = $5)
          AND ($6::text IS NULL OR t.title ILIKE '%' || $6 || '%')
          AND ($7::bool IS NULL OR ($7 = true AND t.due_date < NOW() AND t.status != 'done'))
        ORDER BY t.position ASC, t.created_at DESC
        LIMIT $8 OFFSET $9
        "#,
        claims.sub,
        query.project_id,
        query.assignee_id,
        query.status.as_ref().map(|s| format!("{:?}", s).to_lowercase()),
        query.priority.as_ref().map(|p| format!("{:?}", p).to_lowercase()),
        query.search,
        query.is_overdue,
        per_page as i64,
        offset
    )
    .fetch_all(pool.get_ref())
    .await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "tasks": tasks,
        "page": page,
        "per_page": per_page,
    })))
}

pub async fn create_task(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    body: web::Json<CreateTaskRequest>,
) -> Result<HttpResponse, AppError> {
    body.validate()?;
    let claims = require_auth(&req)?;

    // Verify user is project member
    let is_member: bool = sqlx::query_scalar!(
        "SELECT EXISTS(SELECT 1 FROM project_members WHERE project_id = $1 AND user_id = $2)",
        body.project_id, claims.sub
    )
    .fetch_one(pool.get_ref())
    .await?
    .unwrap_or(false);

    if !is_member {
        return Err(AppError::Forbidden("Not a project member".to_string()));
    }

    // Validate parent task if specified
    if let Some(parent_id) = body.parent_task_id {
        let parent_exists: bool = sqlx::query_scalar!(
            "SELECT EXISTS(SELECT 1 FROM tasks WHERE id = $1 AND project_id = $2)",
            parent_id, body.project_id
        )
        .fetch_one(pool.get_ref())
        .await?
        .unwrap_or(false);

        if !parent_exists {
            return Err(AppError::BadRequest("Parent task not found in this project".to_string()));
        }
    }

    // Get max position
    let max_position: i32 = sqlx::query_scalar!(
        "SELECT COALESCE(MAX(position), 0) FROM tasks WHERE project_id = $1 AND status = $2",
        body.project_id,
        body.status.as_ref().map(|_| "todo").unwrap_or("todo")
    )
    .fetch_one(pool.get_ref())
    .await?
    .unwrap_or(0);

    let task_id = Uuid::new_v4();
    let labels = body.labels.clone().unwrap_or_default();

    sqlx::query!(
        r#"
        INSERT INTO tasks (id, title, description, project_id, parent_task_id,
                          assignee_id, creator_id, status, priority, labels,
                          due_date, start_date, estimated_hours, position)
        VALUES ($1, $2, $3, $4, $5, $6, $7,
                COALESCE($8::task_status, 'todo'::task_status),
                COALESCE($9::task_priority, 'medium'::task_priority),
                $10, $11, $12, $13, $14)
        "#,
        task_id,
        body.title,
        body.description,
        body.project_id,
        body.parent_task_id,
        body.assignee_id,
        claims.sub,
        body.status.as_ref().map(|s| format!("{:?}", s).to_lowercase()) as Option<String>,
        body.priority.as_ref().map(|p| format!("{:?}", p).to_lowercase()) as Option<String>,
        &labels as &[String],
        body.due_date,
        body.start_date,
        body.estimated_hours,
        max_position + 1
    )
    .execute(pool.get_ref())
    .await?;

    // Add dependencies
    if let Some(depends_on) = &body.depends_on {
        for dep_id in depends_on {
            // Check for circular dependencies
            let creates_cycle = check_dependency_cycle(pool.get_ref(), task_id, *dep_id).await?;
            if creates_cycle {
                return Err(AppError::BadRequest("Circular dependency detected".to_string()));
            }

            sqlx::query!(
                "INSERT INTO task_dependencies (task_id, depends_on_id) VALUES ($1, $2) ON CONFLICT DO NOTHING",
                task_id, dep_id
            )
            .execute(pool.get_ref())
            .await?;
        }
    }

    Ok(HttpResponse::Created().json(serde_json::json!({ "id": task_id, "message": "Task created" })))
}

pub async fn move_task(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<Uuid>,
    body: web::Json<MoveTaskRequest>,
) -> Result<HttpResponse, AppError> {
    let claims = require_auth(&req)?;
    let task_id = path.into_inner();

    // Verify the task exists and user has access
    let task = sqlx::query!(
        "SELECT t.id, t.project_id, t.status::text as status FROM tasks t WHERE t.id = $1",
        task_id
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::NotFound("Task not found".to_string()))?;

    let is_member: bool = sqlx::query_scalar!(
        "SELECT EXISTS(SELECT 1 FROM project_members WHERE project_id = $1 AND user_id = $2)",
        task.project_id, claims.sub
    )
    .fetch_one(pool.get_ref())
    .await?
    .unwrap_or(false);

    if !is_member {
        return Err(AppError::Forbidden("Not authorized".to_string()));
    }

    // Update position in target status column
    let new_position = body.position.unwrap_or_else(|| {
        // Default: end of list
        999999
    });

    let new_status = format!("{:?}", body.status).to_lowercase();
    let completed_at = if body.status == TaskStatus::Done {
        Some(chrono::Utc::now())
    } else {
        None
    };

    sqlx::query!(
        r#"
        UPDATE tasks
        SET status = $1::task_status,
            position = $2,
            completed_at = $3,
            updated_at = NOW()
        WHERE id = $4
        "#,
        new_status,
        new_position,
        completed_at,
        task_id
    )
    .execute(pool.get_ref())
    .await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "message": "Task moved",
        "new_status": body.status,
        "position": new_position,
    })))
}

async fn check_dependency_cycle(
    pool: &PgPool,
    task_id: Uuid,
    depends_on_id: Uuid,
) -> Result<bool, AppError> {
    // BFS to check if task_id is reachable from depends_on_id
    let creates_cycle: bool = sqlx::query_scalar!(
        r#"
        WITH RECURSIVE dep_chain AS (
            SELECT depends_on_id FROM task_dependencies WHERE task_id = $1
            UNION ALL
            SELECT td.depends_on_id FROM task_dependencies td
            JOIN dep_chain dc ON td.task_id = dc.depends_on_id
        )
        SELECT EXISTS(SELECT 1 FROM dep_chain WHERE depends_on_id = $2)
        "#,
        depends_on_id, task_id
    )
    .fetch_one(pool)
    .await?
    .unwrap_or(false);

    Ok(creates_cycle)
}
```

---

## 5. Kanban Board Handler (`src/handlers/kanban.rs`)

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use sqlx::PgPool;
use uuid::Uuid;
use std::collections::HashMap;

use crate::errors::AppError;
use crate::middleware::auth::require_auth;
use crate::models::task::TaskStatus;

pub async fn get_kanban_board(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<Uuid>,
) -> Result<HttpResponse, AppError> {
    let claims = require_auth(&req)?;
    let project_id = path.into_inner();

    // Verify membership
    let is_member: bool = sqlx::query_scalar!(
        "SELECT EXISTS(SELECT 1 FROM project_members WHERE project_id = $1 AND user_id = $2)",
        project_id, claims.sub
    )
    .fetch_one(pool.get_ref())
    .await?
    .unwrap_or(false);

    if !is_member {
        return Err(AppError::Forbidden("Not a project member".to_string()));
    }

    // Get all tasks grouped by status
    let tasks = sqlx::query!(
        r#"
        SELECT t.id, t.title, t.description, t.status::text, t.priority::text,
               t.labels, t.due_date, t.estimated_hours, t.position,
               t.is_blocked, t.assignee_id,
               u.username as assignee_username, u.avatar_url as assignee_avatar,
               COUNT(DISTINCT c.id) as comments_count,
               COUNT(DISTINCT a.id) as attachments_count,
               COUNT(DISTINCT st.id) as subtask_count,
               COUNT(DISTINCT st.id) FILTER (WHERE st.status = 'done') as completed_subtask_count
        FROM tasks t
        LEFT JOIN users u ON t.assignee_id = u.id
        LEFT JOIN task_comments c ON t.id = c.task_id
        LEFT JOIN task_attachments a ON t.id = a.task_id
        LEFT JOIN tasks st ON st.parent_task_id = t.id
        WHERE t.project_id = $1 AND t.parent_task_id IS NULL
        GROUP BY t.id, u.username, u.avatar_url
        ORDER BY t.position ASC
        "#,
        project_id
    )
    .fetch_all(pool.get_ref())
    .await?;

    // Group by status
    let columns: Vec<&str> = vec!["backlog", "todo", "in_progress", "in_review", "done", "cancelled"];
    let mut board: HashMap<String, Vec<_>> = HashMap::new();
    
    for col in &columns {
        board.insert(col.to_string(), Vec::new());
    }

    for task in tasks {
        let status = task.status.clone().unwrap_or("todo".to_string());
        board.entry(status).or_default().push(task);
    }

    // Get column stats
    let column_stats = sqlx::query!(
        r#"
        SELECT status::text, COUNT(*) as count
        FROM tasks
        WHERE project_id = $1 AND parent_task_id IS NULL
        GROUP BY status
        "#,
        project_id
    )
    .fetch_all(pool.get_ref())
    .await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "project_id": project_id,
        "columns": columns,
        "tasks": board,
        "stats": column_stats,
    })))
}
```

---

## 6. Time Tracking Handler (`src/handlers/time_tracking.rs`)

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use sqlx::PgPool;
use uuid::Uuid;

use crate::errors::AppError;
use crate::middleware::auth::require_auth;
use crate::models::time_entry::{LogTimeRequest, StartTimerRequest};

pub async fn log_time(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<Uuid>,
    body: web::Json<LogTimeRequest>,
) -> Result<HttpResponse, AppError> {
    let claims = require_auth(&req)?;
    let task_id = path.into_inner();

    if body.hours <= 0.0 || body.hours > 24.0 {
        return Err(AppError::BadRequest("Hours must be between 0 and 24".to_string()));
    }

    let logged_date = body.logged_date.unwrap_or_else(|| chrono::Utc::now().date_naive());
    let entry_id = Uuid::new_v4();

    sqlx::query!(
        r#"
        INSERT INTO time_entries (id, task_id, user_id, hours, description, logged_date)
        VALUES ($1, $2, $3, $4, $5, $6)
        "#,
        entry_id, task_id, claims.sub, body.hours, body.description, logged_date
    )
    .execute(pool.get_ref())
    .await?;

    // Get total hours for this task
    let total_hours: f64 = sqlx::query_scalar!(
        "SELECT COALESCE(SUM(hours), 0) FROM time_entries WHERE task_id = $1",
        task_id
    )
    .fetch_one(pool.get_ref())
    .await?
    .unwrap_or(0.0);

    Ok(HttpResponse::Created().json(serde_json::json!({
        "entry_id": entry_id,
        "hours_logged": body.hours,
        "total_hours_on_task": total_hours,
    })))
}

pub async fn get_task_time_entries(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<Uuid>,
) -> Result<HttpResponse, AppError> {
    require_auth(&req)?;
    let task_id = path.into_inner();

    let entries = sqlx::query!(
        r#"
        SELECT te.id, te.task_id, te.user_id, te.hours, te.description,
               te.logged_date, te.created_at, u.username
        FROM time_entries te
        JOIN users u ON te.user_id = u.id
        WHERE te.task_id = $1
        ORDER BY te.logged_date DESC, te.created_at DESC
        "#,
        task_id
    )
    .fetch_all(pool.get_ref())
    .await?;

    let total_hours: f64 = entries.iter().map(|e| e.hours).sum();

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "entries": entries,
        "total_hours": total_hours,
    })))
}

pub async fn get_project_time_report(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<Uuid>,
) -> Result<HttpResponse, AppError> {
    require_auth(&req)?;
    let project_id = path.into_inner();

    let report = sqlx::query!(
        r#"
        SELECT u.username, u.id as user_id,
               SUM(te.hours) as total_hours,
               COUNT(DISTINCT te.task_id) as tasks_worked_on
        FROM time_entries te
        JOIN tasks t ON te.task_id = t.id
        JOIN users u ON te.user_id = u.id
        WHERE t.project_id = $1
        GROUP BY u.id, u.username
        ORDER BY total_hours DESC
        "#,
        project_id
    )
    .fetch_all(pool.get_ref())
    .await?;

    let total: f64 = report.iter().filter_map(|r| r.total_hours).sum();

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "project_id": project_id,
        "by_user": report,
        "total_hours": total,
    })))
}
```

---

## 7. Main Application (`src/main.rs`)

```rust
use actix_cors::Cors;
use actix_web::{middleware::Logger, web, App, HttpServer};
use dotenv::dotenv;
use sqlx::postgres::PgPoolOptions;
use std::env;

mod errors;
mod handlers;
mod middleware;
mod models;
mod services;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init_from_env(env_logger::Env::new().default_filter_or("info"));

    let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let host = env::var("HOST").unwrap_or_else(|_| "127.0.0.1".to_string());
    let port = env::var("PORT").unwrap_or_else(|_| "8080".to_string());

    let pool = PgPoolOptions::new()
        .max_connections(15)
        .connect(&database_url)
        .await
        .expect("Failed to create pool");

    log::info!("Starting Task Management API at http://{}:{}", host, port);

    HttpServer::new(move || {
        App::new()
            .wrap(Logger::default())
            .wrap(Cors::permissive())
            .app_data(web::Data::new(pool.clone()))
            .service(
                web::scope("/api")
                    // Projects
                    .service(
                        web::scope("/projects")
                            .route("", web::get().to(handlers::projects::list_projects))
                            .route("", web::post().to(handlers::projects::create_project))
                            .route("/{id}", web::get().to(handlers::projects::get_project))
                            .route("/{id}", web::put().to(handlers::projects::update_project))
                            .route("/{id}", web::delete().to(handlers::projects::delete_project))
                            .route("/{id}/members", web::post().to(handlers::projects::add_member))
                            .route("/{id}/kanban", web::get().to(handlers::kanban::get_kanban_board))
                            .route("/{id}/time-report", web::get().to(handlers::time_tracking::get_project_time_report))
                    )
                    // Tasks
                    .service(
                        web::scope("/tasks")
                            .route("", web::get().to(handlers::tasks::list_tasks))
                            .route("", web::post().to(handlers::tasks::create_task))
                            .route("/{id}", web::get().to(handlers::tasks::get_task))
                            .route("/{id}", web::put().to(handlers::tasks::update_task))
                            .route("/{id}", web::delete().to(handlers::tasks::delete_task))
                            .route("/{id}/move", web::post().to(handlers::tasks::move_task))
                            .route("/{id}/assign", web::post().to(handlers::tasks::assign_task))
                            // Comments
                            .route("/{id}/comments", web::get().to(handlers::comments::list_comments))
                            .route("/{id}/comments", web::post().to(handlers::comments::add_comment))
                            // Time tracking
                            .route("/{id}/time", web::get().to(handlers::time_tracking::get_task_time_entries))
                            .route("/{id}/time", web::post().to(handlers::time_tracking::log_time))
                            // Attachments
                            .route("/{id}/attachments", web::post().to(handlers::tasks::upload_attachment))
                    )
            )
    })
    .bind(format!("{}:{}", host, port))?
    .run()
    .await
}
```

---

## 8. Database Schema

```sql
CREATE TYPE project_status AS ENUM ('active', 'completed', 'archived', 'on_hold');
CREATE TYPE member_role AS ENUM ('owner', 'admin', 'member', 'viewer');
CREATE TYPE task_status AS ENUM ('backlog', 'todo', 'in_progress', 'in_review', 'done', 'cancelled');
CREATE TYPE task_priority AS ENUM ('low', 'medium', 'high', 'urgent');

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    color VARCHAR(7) NOT NULL DEFAULT '#3B82F6',
    owner_id UUID NOT NULL REFERENCES users(id),
    status project_status NOT NULL DEFAULT 'active',
    due_date TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE project_members (
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role member_role NOT NULL DEFAULT 'member',
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (project_id, user_id)
);

CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(200) NOT NULL,
    description TEXT,
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    parent_task_id UUID REFERENCES tasks(id) ON DELETE CASCADE,
    assignee_id UUID REFERENCES users(id) ON DELETE SET NULL,
    creator_id UUID NOT NULL REFERENCES users(id),
    status task_status NOT NULL DEFAULT 'todo',
    priority task_priority NOT NULL DEFAULT 'medium',
    labels TEXT[] NOT NULL DEFAULT '{}',
    due_date TIMESTAMPTZ,
    start_date TIMESTAMPTZ,
    estimated_hours DOUBLE PRECISION,
    position INTEGER NOT NULL DEFAULT 0,
    is_blocked BOOLEAN NOT NULL DEFAULT false,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tasks_project ON tasks(project_id);
CREATE INDEX idx_tasks_assignee ON tasks(assignee_id);
CREATE INDEX idx_tasks_status ON tasks(project_id, status);
CREATE INDEX idx_tasks_due ON tasks(due_date) WHERE due_date IS NOT NULL;

CREATE TABLE task_dependencies (
    task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    depends_on_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    dependency_type VARCHAR(20) NOT NULL DEFAULT 'blocks',
    PRIMARY KEY (task_id, depends_on_id)
);

CREATE TABLE task_comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    content TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE time_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    hours DOUBLE PRECISION NOT NULL CHECK (hours > 0 AND hours <= 24),
    description TEXT,
    logged_date DATE NOT NULL DEFAULT CURRENT_DATE,
    started_at TIMESTAMPTZ,
    ended_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE task_attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    uploader_id UUID NOT NULL REFERENCES users(id),
    filename VARCHAR(255) NOT NULL,
    original_name VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    size_bytes BIGINT NOT NULL,
    url TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## สรุป Part 085

ใน Part นี้เราได้สร้าง Task Management API ที่สมบูรณ์ด้วย:
1. **Projects & Members** พร้อม role-based access
2. **Tasks** พร้อม priority, labels, due dates
3. **Kanban Board** ที่ group tasks ตาม status columns
4. **Task Dependencies** พร้อม cycle detection
5. **Time Tracking** พร้อม per-task และ per-project reports
6. **Subtasks** พร้อม parent-child relationships
7. **Comments & Attachments** สำหรับ tasks

ใน **Part 086** เราจะสร้าง **Notification Service** แบบ Multi-channel

---

*[← Part 084: URL Shortener](../part_084/README.md) | [Part 086: Notification Service →](../part_086/README.md)*
