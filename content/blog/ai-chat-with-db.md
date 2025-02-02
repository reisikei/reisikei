---
title: "Chat with a Database via LLMs"
date: 2025-02-03
draft: false
cover:
  image: "https://socialify.git.ci/jalcocert/streamlit-multichat/image?description=1&font=Inter&language=1&name=1&stargazers=1&theme=Auto"
  alt: "Langchain chat with DB" # alt text
  caption: "Using LLMs to chat with databases" # display caption under cover
tags: ["Gen-AI","Python","Dev"]
description: 'How to use Langchain to chat with a DB. How I prepared for a tech talk with AI.'
summary: 'Chat with DB Tech Talk'
url: 'langchain-chat-with-database'
---

In the end of the last year I discoverd: 

{{< cards >}}
  {{< card link="https://jalcocert.github.io/JAlcocerT/how-to-chat-with-your-data" title="Youtube Summarizer" image="/blog_img/GenAI/yt-summaries/yt-summaries-groq.png" subtitle="With Groq API" >}}
  {{< card link="https://github.com/JAlcocerT/Data-Chat" title="Data Chat" image="/blog_img/apps/gh-jalcocert.svg" subtitle="Source Code for DB Chat with Langchain" >}}
{{< /cards >}}

{{< callout type="info" >}}
I am creating a public repo with all sort if interesting RAG libraries to **[Chat over Data](https://github.com/JAlcocerT/Data-Chat)** 💻 
{{< /callout >}}

## Chat with Your Database Using LangChain

Learning SQL can be challenging. 

But what if you could use AI to simplify your data analysis tasks?

That's where **LangChain comes in**, bridging the gap between natural language and your database.

This post explores how to use LangChain with Python to chat with your database.

{{< callout type="info" >}}
The complete, working code is available in my [data-chat repository](https://github.com/JAlcocerT/Data-Chat/tree/main/LangChain/ChatWithDB).
{{< /callout >}}

[![Open in Google Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/JAlcocerT/Data-Chat/blob/main/LangChain/ChatWithDB/test_langchainChatDB.ipynb)

## Setting Up Your Database (MySQL Example)

This example uses **MySQL**, you can also try SQLite.

{{< details title="MySQL Installation CLI" closed="true" >}}

```bash
sudo apt update
sudo apt install mysql-server -y
sudo systemctl start mysql
sudo systemctl status mysql
sudo systemctl enable mysql  # Start MySQL on boot

sudo mysql -u root -p
```

```bash
# mysql --version  # Output: mysql Ver 8.0.40-0ubuntu0.24.04.1 for Linux on x86_64 ((Ubuntu))
```

{{< /details >}}

I would recommend to go with the **container way of installing the DB**:

{{< details title="MySQL with Docker | Recommended" closed="true" >}}

Once, Docker/Podman are installed, just:

```sh

```

```sh
# mysql --version  # Output: mysql Ver 8.0.40-0ubuntu0.24.04.1 for Linux on x86_64 ((Ubuntu))
```

{{< /details >}}

## Loading the Chinook Database

We'll use the **Chinook database as a sample**.


{{< callout type="info" >}}
Be creative, you can apply it to other DB's! Like the one of [this project](https://jalcocert.github.io/RPi/posts/rpi-iot-dht1122-mongo/) 
{{< /callout >}}

{{< details title="Loading the Chinook Database" closed="true" >}}

You can download the Chinook database from its [GitHub releases](https://github.com/lerocha/chinook-database/releases).

```sql
CREATE DATABASE chinook;
USE chinook;
SOURCE Chinook_MySql.sql;  # Or the name of your SQL file

SHOW TABLES;
SELECT * FROM Album LIMIT 10;
DESCRIBE Artist;
```

{{< /details >}}

Now that the database is loaded, let's explore its schema.

```sql
SELECT 
    TABLE_NAME AS `Table`, 
    COLUMN_NAME AS `Column`, 
    DATA_TYPE AS `Data Type`, 
    CHARACTER_MAXIMUM_LENGTH AS `Max Length`, 
    IS_NULLABLE AS `Nullable`, 
    COLUMN_KEY AS `Key`, 
    COLUMN_DEFAULT AS `Default Value`
FROM 
    INFORMATION_SCHEMA.COLUMNS
WHERE 
    TABLE_SCHEMA = 'chinook'
ORDER BY 
    TABLE_NAME, ORDINAL_POSITION;
```

Here's a MermaidJS Entity-Relationship (ER) diagram visualizing the **Chinook database schema**:

```mermaid
erDiagram
    Album {
        int AlbumId PK
        varchar Title
        int ArtistId FK
    }
    Artist {
        int ArtistId PK
        varchar Name
    }
    Customer {
        int CustomerId PK
        varchar FirstName
        varchar LastName
        varchar Company
        varchar Address
        varchar City
        varchar State
        varchar Country
        varchar PostalCode
        varchar Phone
        varchar Fax
        varchar Email
        int SupportRepId FK
    }
    Employee {
        int EmployeeId PK
        varchar LastName
        varchar FirstName
        varchar Title
        int ReportsTo FK
        datetime BirthDate
        datetime HireDate
        varchar Address
        varchar City
        varchar State
        varchar Country
        varchar PostalCode
        varchar Phone
        varchar Fax
        varchar Email
    }
    Genre {
        int GenreId PK
        varchar Name
    }
    Invoice {
        int InvoiceId PK
        int CustomerId FK
        datetime InvoiceDate
        varchar BillingAddress
        varchar BillingCity
        varchar BillingState
        varchar BillingCountry
        varchar BillingPostalCode
        decimal Total
    }
    InvoiceLine {
        int InvoiceLineId PK
        int InvoiceId FK
        int TrackId FK
        decimal UnitPrice
        int Quantity
    }
    MediaType {
        int MediaTypeId PK
        varchar Name
    }
    Playlist {
        int PlaylistId PK
        varchar Name
    }
    PlaylistTrack {
        int PlaylistId PK
        int TrackId PK
    }
    Track {
        int TrackId PK
        varchar Name
        int AlbumId FK
        int MediaTypeId FK
        int GenreId FK
        varchar Composer
        int Milliseconds
        int Bytes
        decimal UnitPrice
    }

    %% Relationships
    Album ||--o{ Artist : "belongs to"
    Track ||--o{ Album : "belongs to"
    Track ||--o{ MediaType : "uses"
    Track ||--o{ Genre : "categorized as"
    PlaylistTrack ||--o{ Playlist : "contains"
    PlaylistTrack ||--o{ Track : "contains"
    Invoice ||--o{ Customer : "issued to"
    InvoiceLine ||--o{ Invoice : "part of"
    InvoiceLine ||--o{ Track : "tracks"
    Customer ||--o{ Employee : "assisted by"
    Employee ||--o{ Employee : "reports to"
```

## LangChain Setup

Let's set up LangChain to interact with our database.

```bash
python3 -m venv datachat_venv  # Create the virtual environment (Linux)
# python -m venv datachat_venv  # Create the virtual environment (Windows)

# datachat_venv\Scripts\activate  # Activate the virtual environment (Windows)
source datachat_venv/bin/activate  # Activate the virtual environment (Linux)

pip install -r requirements.txt
```

{{< details title="Loading Your OpenAI API Key" closed="true" >}}

```bash
# source .env  # If you're using a .env file

# export OPENAI_API_KEY="your-api-key-here"  # Linux/macOS
# set OPENAI_API_KEY=your-api-key-here  # Windows
# $env:OPENAI_API_KEY="your-api-key-here" #Powershell

# echo $OPENAI_API_KEY  # Verify the key
```

{{< /details >}}

## Integrating LangChain with the Database

I've successfully replicated the code, using:

*   Chinook database version 1.4.5
*   Python 3.12.3 (x86) and virtual environments
*   MySQL 8.0

For easier setup and deployment, you can use Docker.  Here's a sample `docker-compose.yml` file (available in the repository):

```yaml
# ... (docker-compose.yml content)
```

```bash
# Example commands for interacting with the MySQL container:
# docker exec -it mysql_db bash
# docker exec -it mysql_db mysql -u myuser -p chinook
```

{{< callout type="info" >}}
For containerization, you'll need to have [Docker installed](https://jalcocert.github.io/JAlcocerT/docs/dev/dev-interesting-it-concepts/#containers).  This also prepares you for [self-hosting](https://jalcocert.github.io/Linux/docs/linux__cloud/selfhosting/).
{{< /callout >}}

---

## Frequently Asked Questions

{{< details title="More Useful Database Tools" closed="true" >}}

*   [ChartDB](https://github.com/chartdb/chartdb) - Database diagram editor.
*   [SQLiteViz](https://github.com/lana-k/sqliteviz)
*   [SQLiteBrowser](https://github.com/sqlitebrowser/sqlitebrowser)

{{< /details >}}



---

## FAQ

### Preparing a Tech Talk with AI

1. Diagrams as a Code
2. PPT with LLMs