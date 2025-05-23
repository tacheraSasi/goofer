# Quickstart

This quickstart guide will help you set up a simple Goofer ORM project with a User and Post model, demonstrating basic CRUD operations and relationships.

## Initialize a new Go project

If you don't have a Go project yet, initialize one using Go modules:

```shell
mkdir goofer-demo && cd goofer-demo
go mod init goofer-demo
```

## Install Goofer ORM

Install the Goofer ORM package:

```shell
go get github.com/gooferOrm/goofer
```

You'll also need a database driver. For this quickstart, we'll use SQLite:

```shell
go get github.com/mattn/go-sqlite3
```

## Create your models

Create a file named `models.go` with the following content:

```go
package main

import (
	"time"
)

// User entity
type User struct {
	ID        uint      `orm:"primaryKey;autoIncrement" validate:"required"`
	Name      string    `orm:"type:varchar(255);notnull" validate:"required"`
	Email     string    `orm:"unique;type:varchar(255);notnull" validate:"required,email"`
	CreatedAt time.Time `orm:"type:timestamp;default:CURRENT_TIMESTAMP"`
	Posts     []Post    `orm:"relation:OneToMany;foreignKey:UserID"`
}

// TableName returns the table name for the User entity
func (User) TableName() string {
	return "users"
}

// Post entity
type Post struct {
	ID        uint      `orm:"primaryKey;autoIncrement" validate:"required"`
	Title     string    `orm:"type:varchar(255);notnull" validate:"required"`
	Content   string    `orm:"type:text" validate:"required"`
	UserID    uint      `orm:"index;notnull" validate:"required"`
	CreatedAt time.Time `orm:"type:timestamp;default:CURRENT_TIMESTAMP"`
	User      *User     `orm:"relation:ManyToOne;foreignKey:UserID"`
}

// TableName returns the table name for the Post entity
func (Post) TableName() string {
	return "posts"
}
```

## Create your main application

Create a file named `main.go` with the following content:

```go
package main

import (
	"database/sql"
	"fmt"
	"log"
	"time"

	_ "github.com/mattn/go-sqlite3"

	"github.com/gooferOrm/goofer/pkg/dialect"
	"github.com/gooferOrm/goofer/pkg/repository"
	"github.com/gooferOrm/goofer/pkg/schema"
)

func main() {
	// Open SQLite database
	db, err := sql.Open("sqlite3", ":memory:")
	if err != nil {
		log.Fatalf("Failed to open database: %v", err)
	}
	defer db.Close()

	// Create dialect
	sqliteDialect := &dialect.SQLiteDialect{}

	// Register entities
	if err := schema.Registry.RegisterEntity(User{}); err != nil {
		log.Fatalf("Failed to register User entity: %v", err)
	}
	if err := schema.Registry.RegisterEntity(Post{}); err != nil {
		log.Fatalf("Failed to register Post entity: %v", err)
	}

	// Create tables
	userMeta, _ := schema.Registry.GetEntityMetadata(schema.GetEntityType(User{}))
	postMeta, _ := schema.Registry.GetEntityMetadata(schema.GetEntityType(Post{}))

	// Create tables
	_, err = db.Exec(sqliteDialect.CreateTableSQL(userMeta))
	if err != nil {
		log.Fatalf("Failed to create users table: %v", err)
	}

	_, err = db.Exec(sqliteDialect.CreateTableSQL(postMeta))
	if err != nil {
		log.Fatalf("Failed to create posts table: %v", err)
	}

	// Create repositories
	userRepo := repository.NewRepository[User](db, sqliteDialect)
	postRepo := repository.NewRepository[Post](db, sqliteDialect)

	// Create a user
	user := &User{
		Name:  "John Doe",
		Email: "john@example.com",
	}

	// Save the user
	if err := userRepo.Save(user); err != nil {
		log.Fatalf("Failed to save user: %v", err)
	}

	fmt.Printf("Created user with ID: %d\n", user.ID)

	// Create a post
	post := &Post{
		Title:   "Hello, Goofer ORM!",
		Content: "This is my first post using Goofer ORM.",
		UserID:  user.ID,
	}

	// Save the post
	if err := postRepo.Save(post); err != nil {
		log.Fatalf("Failed to save post: %v", err)
	}

	fmt.Printf("Created post with ID: %d\n", post.ID)

	// Find the user by ID
	foundUser, err := userRepo.FindByID(user.ID)
	if err != nil {
		log.Fatalf("Failed to find user: %v", err)
	}

	fmt.Printf("Found user: %s (%s)\n", foundUser.Name, foundUser.Email)

	// Find posts by user ID
	posts, err := postRepo.Find().Where("user_id = ?", user.ID).All()
	if err != nil {
		log.Fatalf("Failed to find posts: %v", err)
	}

	fmt.Printf("Found %d posts by user %s:\n", len(posts), foundUser.Name)
	for _, p := range posts {
		fmt.Printf("- %s: %s\n", p.Title, p.Content)
	}

	// Update the user
	foundUser.Name = "Jane Doe"
	if err := userRepo.Save(foundUser); err != nil {
		log.Fatalf("Failed to update user: %v", err)
	}

	fmt.Printf("Updated user name to: %s\n", foundUser.Name)

	// Transaction example
	err = userRepo.Transaction(func(txRepo *repository.Repository[User]) error {
		// Create a new user in the transaction
		newUser := &User{
			Name:  "Transaction User",
			Email: "tx@example.com",
		}

		// Save the user in the transaction
		if err := txRepo.Save(newUser); err != nil {
			return err
		}

		fmt.Printf("Created user in transaction with ID: %d\n", newUser.ID)

		// Uncomment to simulate an error and rollback the transaction
		// return errors.New("simulated error")

		return nil
	})

	if err != nil {
		log.Printf("Transaction failed: %v", err)
	} else {
		fmt.Println("Transaction committed successfully")
	}
}
```

## Run your application

Run your application with:

```shell
go mod tidy
go run .
```

You should see output similar to:

```
Created user with ID: 1
Created post with ID: 1
Found user: John Doe (john@example.com)
Found 1 posts by user John Doe:
- Hello, Goofer ORM!: This is my first post using Goofer ORM.
Updated user name to: Jane Doe
Created user in transaction with ID: 2
Transaction committed successfully
```

## Next steps

This quickstart demonstrated the basics of Goofer ORM. Here are some next steps to explore:

- Learn about [entity relationships](../examples/relationships) in depth
- Explore [migrations](../reference/features/migrations) for schema evolution
- Check out the [CLI](../cli) for automation
- Dive into [advanced queries](../examples/queries) for more complex data access
- Learn about [validation](../reference/features/validation) for data integrity

Goofer ORM is designed to make working with databases in Go a pleasant experience. Enjoy building with type-safe, relationship-aware database access!
