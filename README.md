# spec
**OpenAPI Documentation that stays out of the way**

The easiest way to document a Chi API.

No handler changes. No generators.

**To install**
```bash
go get github.com/mainanick/spec
```
**Quickstart**
```go
package main

import (
    "net/http"

    "github.com/go-chi/chi/v5"
    "github.com/mainanick/spec"
)

func main() {
    r := spec.NewRouter(
        spec.Title("My API"),
        spec.Version("1.0.0"),
    )

    r.Get("/users/{id}", GetUser).
        Returns(200, User{})

    r.Post("/users", CreateUser).
        Body(CreateUserInput{}).
        Returns(201, User{})

    http.ListenAndServe(":3000", r)
}

```
Handlers stay completely normal:

```go
func GetUser(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    // your code
}

```
**Typical Usage**

```go

r := spec.NewRouter(
    spec.Title("My API"),
    spec.Version("1.0.0"),
spec.Server("https://api.example.com/v1"),
)

r.Tag("Users", "User operations")

r.Route("/users", func(r spec.Router) {
    r.Tags("Users")
    r.Security("BearerAuth")

    r.Post("/", CreateUser).
        Summary("Create a user").
        Body(CreateUserInput{}).
        Returns(201, User{}).
        Returns(400, Error{})

    r.Get("/{id}", GetUser).
        PathParam("id", spec.Integer).
        Returns(200, User{}).
        Returns(404, Error{})
})

```

**Existing Chi Router**
Already have a Chi router? Just wrap it:

```go
r := chi.NewRouter()
r.Use(middleware.Logger)
r.Use(middleware.Recoverer)

api := spec.Wrap(r,
    spec.Title("My API"),
    spec.Version("1.0.0"),
)

api.Get("/users/{id}", GetUser).
    Returns(200, User{})

```

**Documenting Structs**

spec builds schemas directly from your Go structs. You rarely need to write extra documentation.

```go
type CreateUserInput struct {
    Email    string  `json:"email" doc:"User's email address" example:"alice@example.com"`
    Name     string  `json:"name" doc:"Full name"`
    Age      *int    `json:"age,omitempty" doc:"Optional age"`
    Role     string  `json:"role" enum:"admin,user,guest"`
    Password string  `json:"-"` // ignored
}

```

**Advanced**

```go
r := spec.NewRouter(
    spec.Title("My API"),
    spec.Version("1.0.0"),
    spec.Description("A clean API"),
    spec.Server("https://api.example.com/v1"),
    spec.Server("https://staging.example.com/v1", spec.Description("Staging")),
)

r.SecurityScheme("BearerAuth", spec.BearerAuth("JWT"))
r.SecurityScheme("ApiKey", spec.APIKey("header", "X-API-Key"))

r.Tag("Users", "User management")
r.Tag("Admin", "Administrative operations")

r.Route("/users", func(r spec.Router) {
    r.Tags("Users")
    r.Security("BearerAuth")

    r.Post("/", CreateUser).
        Summary("Create a user").
        Description("Creates a new user account").
        Body(CreateUserInput{}).
        Example(CreateUserInput{Email: "alice@example.com"}).
        Returns(201, User{}).
        Returns(400, Error{}, spec.Description("Validation failed"))

    r.Get("/{id}", GetUser).
        Summary("Get a user").
        PathParam("id", spec.Integer).
        Returns(200, User{}).
        Returns(404, Error{})
})

r.Route("/admin", func(r spec.Router) {
    r.Tags("Admin")
    r.Security("BearerAuth", "ApiKey")

    r.Get("/stats", GetStats).
        Summary("Admin statistics").
        Returns(200, Stats{},
            spec.Header("X-RateLimit-Limit", spec.Integer),
            spec.Header("X-RateLimit-Remaining", spec.Integer),
        )
})

```
**Export**

```go

doc := r.OpenAPI()

// YAML
yaml, err := doc.YAML()

// JSON
json, err := doc.JSON()

// Write to file
err = doc.WriteFile("openapi.yaml")

```

**Serve It**
```go
r.Get("/openapi.yaml", func(w http.ResponseWriter, req *http.Request) {
    w.Header().Set("Content-Type", "application/yaml")
    doc.WriteYAML(w)
})

```
**How it works**

spec is just a thin wrapper around Chi.

- spec.Router embeds chi.Router
- It does not replace Chi
- It does not change your handlers
- It does not add middleware or runtime overhead
- It only records documentation while you register routes.
  
You keep writing Chi exactly the way you already do.
Behind the scenes it looks roughly like this:

```go
type Router struct {
    chi.Router          // embedded — all Chi methods work normally
    doc *Document       // collects OpenAPI metadata
}

func (r *Router) Get(pattern string, h http.HandlerFunc) *Route {
    r.Router.Get(pattern, h)          // normal Chi registration
    return &Route{                    // return a lightweight route object
        method:  "GET",
        pattern: pattern,
        doc:     r.doc,
    }
}

func (rt *Route) Returns(status int, body any) *Route {
    // just stores metadata — zero runtime cost
    rt.doc.addResponse(rt.method, rt.pattern, status, body)
    return rt
}

```

