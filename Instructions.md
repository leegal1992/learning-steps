# LearningSteps API

Welcome to LearningSteps! LearningSteps is a Python FastAPI + PostgreSQL application that helps people track their daily learning journey. This is a reference implementation, deploy this to the cloud!


## Table of Contents

- [🚀 Getting Started](#-getting-started)
- [⚙️ API Endpoints](#-api-endpoints)
- [☁️ Azure CLI for Deployment](#️-azure-cli-for-deployment)
- [📊 Data Schema](#-data-schema)
- [�️ Explore Your Database (Optional)](#️-explore-your-database-optional)
- [🔧 Troubleshooting](#-troubleshooting)

## 🚀 Getting Started

### Prerequisites

- Git installed on your machine
- Docker Desktop installed and running
- VS Code with the Dev Containers extension

### 1. Clone the Repository

1. **Clone this repository** to your local machine:

   ```bash
   git clone https://github.com/YOUR_USERNAME/learningsteps.git
   ```

1. Move into the project directory:

   ```bash
   cd learningsteps
   ```

1. **Open in VS Code**:

   ```bash
   code .
   ```

### 2. Configure Your Environment (.env)

Environment variables live in a `.env` file (which is **git-ignored** so you don't accidentally commit secrets). This repo ships with a template named `.env-sample`.

1. Copy the sample file to create your real `.env`:

   ```bash
   cp .env-sample .env
   ```

### 3. Set Up Your Development Environment

1. **Install the Dev Containers extension** in VS Code (if not already installed)
2. **Reopen in container**: When VS Code detects the `.devcontainer` folder, click "Reopen in Container"
   - Or use Command Palette (`Cmd/Ctrl + Shift + P`): `Dev Containers: Reopen in Container`
3. **Wait for setup**: The API container will automatically install Python, dependencies, and configure your environment.
   The PostgreSQL Database container will also automatically be created.

### 4. Verify the PostgreSQL Database Is Running

In a terminal outside of VS Code, run:

   ```bash
      docker ps
   ```

You should see the postgres service running.

### 5. Run the API

Make sure you are in the root of your project in the terminal (inside VS Code, while container is running):

   ```bash
     ./start.sh
   ```

### 6. Test Everything Works! 🎉

1. **Visit the API docs**: http://localhost:8000/docs
1. **Create your first entry** In the Docs UI Use the POST `/entries` endpoint to create a new journal entry.
1. **View your entries** using the GET `/entries` endpoint to see what you've created!

**🎯 Congratulations! You have a fully functional learning journal API with complete CRUD operations, validation, and logging!**

## ⚙️ API Endpoints

This learning journal API provides complete CRUD functionality:

- **POST /entries** - Create a new journal entry with validation
- **GET /entries** - Retrieve all journal entries with count
- **GET /entries/{entry_id}** - Retrieve a single journal entry by ID
- **PATCH /entries/{entry_id}** - Update an existing journal entry
- **DELETE /entries/{entry_id}** - Delete a specific journal entry
- **DELETE /entries** - Delete all journal entries

All endpoints include proper error handling, data validation using Pydantic models, and structured logging for debugging and monitoring.

## ☁️ Azure CLI for Deployment

The dev container includes the Azure CLI pre-installed, allowing you to deploy your API directly to Azure cloud services. Use the `az` command in your terminal to authenticate with Azure, create resources like App Services and PostgreSQL databases, and deploy your application—all from within your development environment.

## 📊 Data Schema

Each journal entry follows this structure:

| Field       | Type      | Description                                | Validation                   |
|-------------|-----------|--------------------------------------------|------------------------------|
| id          | string    | Unique identifier (UUID)                   | Auto-generated               |
| work        | string    | What did you work on today?                | Required, max 256 characters |
| struggle    | string    | What's one thing you struggled with today? | Required, max 256 characters |
| intention   | string    | What will you study/work on tomorrow?      | Required, max 256 characters |
| created_at  | datetime  | When entry was created                     | Auto-generated UTC           |
| updated_at  | datetime  | When entry was last updated                | Auto-updated UTC             |

## 🗄️ Explore Your Database (Optional)

Want to see your data directly in the database? You can connect to PostgreSQL using VS Code's PostgreSQL extension:

### 1. Install PostgreSQL Extension

1. **Install the PostgreSQL extension** in VS Code (search for "PostgreSQL" by Chris Kolkman)
2. **Restart VS Code** after installation

### 2. Connect to Your Database

1. **Open the PostgreSQL extension** (click the PostgreSQL icon in the sidebar)
2. **Click "Add Connection"** or the "+" button
3. **Enter these connection details**:
   - **Host name**: `postgres`
   - **User name**: `postgres`
   - **Password**: `postgres`
   - **Port**: `5432`
   - **Conection Type**: `Standard/No SSL`
   - **Database**: `learning_journal`
   - **Display name**: `Learning Journal DB` (or any name you prefer)

### 3. Explore Your Data

1. **Expand your connection** in the PostgreSQL panel
2. **Left-click on "Learning Journal DB" to expand**
3. **Right-click on "learning_journal"**
4. **Select "New Query"**
5. **Type this query** to see all your entries:

   ```sql
   SELECT * FROM entries;
   ```

6. **Run the query** to see all your journal data! (Ctrl/Cmd + Enter OR use the PostgreSQL command pallete: Run Query)

You can now explore the database structure, see exactly how your data is stored, and run custom queries to understand PostgreSQL better.

## 🔧 Troubleshooting

**If the API won't start:**

- Make sure the PostgreSQL container is running: `docker ps`
- Check the container logs: `docker logs your-postgres-container-name`
- Restart the database: `docker restart your-postgres-container-name`

**If you can't connect to the database:**

- Verify the `.env` file exists and has the correct DATABASE_URL
- Make sure Docker Desktop is running
- Try restarting the dev container: `Dev Containers: Rebuild Container`

**If the dev container won't open:**

- Ensure Docker Desktop is running
- Install the "Dev Containers" extension in VS Code
- Try: `Dev Containers: Rebuild and Reopen in Container`
