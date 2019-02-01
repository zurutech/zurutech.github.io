---
author: "ridham"
layout: post
did: "blog7"
title:  "Getting started with graphql in GoLang"
slug: "Graphql with GoLang"
date:   2019-02-01 8:00:00
categories: go
img: graphql-go.png
banner: graphql-go.png
tags: coding
description: "Getting started with graphql in GoLang"
---

GraphQL has been a buzzword for last few years after Facebook has made it
open-source and so I tried GraphQL with the Node.js and I completely agree with
all the buzz about the GraphQL, its advantages, and simplicity.

I recently switched to Go-lang for the new project from the Node.js and I
decided to try GraphQL with the Go-lang. There are not many library options with
the Go-lang but I have tried it with this 4 libraries.
[Thunder](https://github.com/samsarahq/thunder),
[graphql](https://github.com/graphql-go/graphql),
[graphql-go](https://github.com/graph-gophers/graphql-go), and
[gqlgen](https://github.com/99designs/gqlgen). And I have to say that
[gqlgen](https://github.com/99designs/gqlgen) is winning all the ground in all
libraries I have tried.

[gqlgen](https://github.com/99designs/gqlgen) is still in the beta with latest
version [v0.4.4](https://github.com/99designs/gqlgen/releases/tag/v0.4.4) at the
time of writing this article and it’s rapidly evolving. You can find their
road-map [here](https://github.com/99designs/gqlgen/projects/1). And as now
[99designs](https://99designs.com/) are officially sponsoring them so we will
see even better development speed for this awesome opensource project.
[vektah](https://github.com/vektah) and [neelance](https://github.com/neelance)
are major contributors of this and [neelance](https://github.com/neelance) also
wrote the [graphql-go](https://github.com/graph-gophers/graphql-go) before.

So let’s dive into the library semantic assuming you have basic
[graphql](https://graphql.org/) knowledge.

<div markdown="1" class="blog-image-container">
![GraphQL with GoLang](https://cdn-images-1.medium.com/max/800/1*_BNR42geWYQPtDojgc8xEQ.png "GraphQL with GoLang"){:class="blog-image"}
</div>

### **Highlights**

As their headline states:

_This is a library for quickly creating strictly typed graphql servers in golang._

I think this is the most promising thing about thislibrary that you will never see
`map[string]interface{}` here as it uses strictly typed approach.

Apart from that, it uses **Schema first Approach**. So you define your
API using the graphql [Schema Definition
Language](http://graphql.org/learn/schema/) and has its own powerful code
generation tools which will auto-generate all of your GraphQL code and you will
just need to implement the core logic of that interface methods.

### Code

As it’s schema-first approach so let’s quickly define a schema. We are taking
job application forum as an example.

```graphql
type Job {
    id: ID!
    name: String!
    description: String!
    location: String!
    createdBy: User!
    createdAt: Timestamp!
    deletedAt: Timestamp
}

type User {
    id: ID!
    name: String!
    email: String!
}

type Application {
    id: ID!
    name: String!
    email: String!
    cvURL: String!
    job: Job!
    createdAt: Timestamp!
}

input NewJob {
    name: String!
    description: String!
    location: String!
    createdByID: String!
}

input NewApplication {
    name: String!
    email:  String!
    jobID: String!
    cvURL: String!
}

type Mutation {
    createJob(input: NewJob!): Job!
    deleteJob(id: ID!): String!
    createApplication(input: NewApplication!): Application!
}

type Query {
    jobs: [Job!]!
    applications(jobID: ID!): [Application!]!
}

scalar Timestamp
```

Now library provide codegen command `gqlgen init` which will generate all the
required files for project files

* gqlgen.yml — The gqlgen config file, knobs for controlling the generated code.
* generated.go — The graphql execution runtime, the bulk of the generated code
* models_gen.go — Generated models required to build the graph. Often you will
override these with models you write yourself. Still very useful for input
types.
* resolver.go — This is where your application code lives. generated.go will call
into this to get the data the user has requested.

As the docs stats if you don’t provide the models definition it will
auto-generate the models but in most of the cases you will write your own model
as you will have more fine-grained control on data types and you can use
third-party structs when needed(Like you can use AWS or any other library’s
Struct for like credential, config, etc).

So let’s override generate `gqlgen.yml` file:

```yaml
schema: schema.graphql

# Let gqlgen know where to put the generated server
exec:
  filename: graph/generated.go
  package: graph

# Let gqlgen know where to the generated models (if any)
model:
  filename: models/inputs.go
  package: models

# Optional, turns on resolver stub generation
resolver:
  filename: resolver.go # where to write them
  type: Resolver  # whats the resolver root implementation type called?

# Tell gqlgen about any existing models you want to reuse for
# graphql. These normally come from the db or a remote api.
models:
  Job:
    model: github.com/ridhamtarpara/go-graphql-jobs/models.Job
  Application:
    model: github.com/ridhamtarpara/go-graphql-jobs/models.Application
  User:
    model: github.com/ridhamtarpara/go-graphql-jobs/models.User
  # Custom scalars point to the name without the Marshal|Unmarshal in front
  Timestamp:
    model: github.com/ridhamtarpara/go-graphql-jobs/models.Timestamp
```

Here we have given the path of all the models. In `schema.graphql` file we have
defined one custom scalar `Timestamp` so we need to tell graphql about how to
Marshal and Unmarshal them.

Here is our `models.go` file where all above structs are defined:

```go
package models

import (
	"time"
	"github.com/99designs/gqlgen/graphql"
	"io"
	"strconv"
	"errors"
)

type Application struct {
	ID          string  `json:"id"`
	Name        string  `json:"name"`
	Email       string  `json:"email"`
	JobId       string  `json:"job"`
	CreatedAt   time.Time  `json:"createdAt"`
}
type Job struct {
	ID          string  `json:"id"`
	Name        string  `json:"name"`
	Description string  `json:"description"`
	Location    string  `json:"location"`
	CreatedBy   string  `json:"createdBy"`
	CreatedAt   time.Time  `json:"createdAt"`
	DeletedAt   *time.Time `json:"deletedAt"`
}

type User struct {
	ID    string `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

func MarshalTimestamp(t time.Time) graphql.Marshaler {
	timestamp := t.Unix()
	if timestamp < 0 {
		timestamp = 0
	}
	return graphql.WriterFunc(func(w io.Writer) {
		io.WriteString(w, strconv.FormatInt(timestamp, 10))
	})
}

func UnmarshalTimestamp(v interface{}) (time.Time, error) {
	if tmpStr, ok := v.(int); ok {
		return time.Unix(int64(tmpStr), 0), nil
	}
	return time.Time{}, errors.New("time should be a unix timestamp")
}
```

Now if you have noticed we haven’t defined any struct for `Timestamp` but only
Marshal and Unmarshal methods for Timestamp. So the library handles this for you
just need to write these two methods.

Another important thing to notice is we haven’t defined a struct for
inputs(`NewJob` and `NewApplication`) too. So as promised by the library it will
be auto-generated in `models/inputs.go` as per the given config in `gqlgen.yml`
file with a notice that you should not edit this file.

```go
// Code generated by github.com/99designs/gqlgen, DO NOT EDIT.

package models

type NewApplication struct {
	Name  string `json:"name"`
	Email string `json:"email"`
	JobID string `json:"jobID"`
	CvURL string `json:"cvURL"`
}
type NewJob struct {
	Name        string `json:"name"`
	Description string `json:"description"`
	Location    string `json:"location"`
	CreatedByID string `json:"createdByID"`
}
```


Now, let’s come to the most important file and that is `generated.go` , which
has all the magic code written its very long file obviously but let’s have a
look at needed code from it.

```go
type Config struct {
	Resolvers  ResolverRoot
	Directives DirectiveRoot
}

type ResolverRoot interface {
	Application() ApplicationResolver
	Job() JobResolver
	Mutation() MutationResolver
	Query() QueryResolver
}

type DirectiveRoot struct {
}
type ApplicationResolver interface {
	Job(ctx context.Context, obj *models.Application) (models.Job, error)
}
type JobResolver interface {
	CreatedBy(ctx context.Context, obj *models.Job) (models.User, error)
}
type MutationResolver interface {
	CreateJob(ctx context.Context, input models.NewJob) (models.Job, error)
	DeleteJob(ctx context.Context, id string) (string, error)
	CreateApplication(ctx context.Context, input models.NewApplication) (models.Application, error)
}
type QueryResolver interface {
	Jobs(ctx context.Context) ([]models.Job, error)
	Applications(ctx context.Context, jobID string) ([]models.Application, error)
}

type executableSchema struct {
	resolvers  ResolverRoot
	directives DirectiveRoot
}
```

Another file generated by gqlgen is `resolver.go` which is the file where you
will put all of your logic. Following is auto-generated stub file(Little edited
for comments, formatting and package change)

```go
//go:generate gorunpkg github.com/99designs/gqlgen

package go_graphql_jobs

import (
	context "context"
	"github.com/ridhamtarpara/go-graphql-jobs/models"
	"github.com/ridhamtarpara/go-graphql-jobs/graph"
)

type Resolver struct{}

func (r *Resolver) Mutation() graph.MutationResolver {
	return &mutationResolver{r}
}
func (r *Resolver) Query() graph.QueryResolver {
	return &queryResolver{r}
}
func (r *Resolver) Application() graph.ApplicationResolver {
	return &applicationResolver{r}
}
func (r *Resolver) Job() graph.JobResolver {
	return &jobResolver{r}
}

// Mutations
type mutationResolver struct{ *Resolver }

func (r *mutationResolver) CreateJob(ctx context.Context, input models.NewJob) (models.Job, error) {
	panic("not implemented")
}
func (r *mutationResolver) DeleteJob(ctx context.Context, id string) (string, error) {
	panic("not implemented")
}
func (r *mutationResolver) CreateApplication(ctx context.Context, input models.NewApplication) (models.Application, error) {
	panic("not implemented")
}

// Queries
type queryResolver struct{ *Resolver }

func (r *queryResolver) Jobs(ctx context.Context) ([]models.Job, error) {
	panic("not implemented")
}
func (r *queryResolver) Applications(ctx context.Context, jobID string) ([]models.Application, error) {
	panic("not implemented")
}

// Resolvers
type jobResolver struct{ *Resolver }

func (r *jobResolver) CreatedBy(ctx context.Context, obj *models.Job) (models.User, error) {
	panic("not implemented")
}

type applicationResolver struct{ *Resolver }

func (r *applicationResolver) Job(ctx context.Context, obj *models.Application) (models.Job, error) {
	panic("not implemented")
}
```

Now all you need to do is write your logic in these methods.

I am writing one sample method to create and get all jobs with the help of
Firebase real-time database. and cutting down the database code for the sake of
he length of this article which you can find
[here](https://github.com/ridhamtarpara/go-graphql-example/tree/feat/blog-1).

```go
func (r *mutationResolver) CreateJob(ctx context.Context, input models.NewJob) (dal.Job, error) {
	jobRepository, err := dal.NewJobFirebaseRepository(r.db.Conn, r.db.Context)
	if err != nil {
		fmt.Printf("firebase error: ", err)
		return dal.Job{}, err
	}

	var jobID string
	if jobID, err = jobRepository.GetID(); err != nil {
		fmt.Printf("jobRepository GetID error: ", err)
		return dal.Job{}, err
	}
	// Create job object from request
	job := dal.Job{
		ID:          jobID,
		Name:        input.Name,
		Description: input.Description,
		Location: input.Location,
		CreatedBy: input.CreatedByID,
		CreatedAt:   time.Now().UTC(),
	}

	// Set the values in the DB
	if err = jobRepository.Insert(job); err != nil {
		fmt.Printf("firebase error: ", err)
		return dal.Job{}, err
	}

	return job, nil
}

func (r *queryResolver) Jobs(ctx context.Context) ([]dal.Job, error) {
	var allJobs []dal.Job

	jobRepository, err := dal.NewJobFirebaseRepository(r.db.Conn, r.db.Context)

	if allJobs, err = jobRepository.GetAll(); err != nil {
		fmt.Printf("firebase error", err)
	}

	return allJobs, nil
}
```


Now let’s try this out and hit the servers and run first mutation

    $ go run server/server.go

<div markdown="1" class="blog-image-container">
![GraphQL with GoLang](https://cdn-images-1.medium.com/max/800/1*UDQ3kseiQmgRBruRDcONAA.png "GraphQL with GoLang"){:class="blog-image"}
</div>

<div markdown="1" class="blog-image-container">
![GraphQL with GoLang](https://cdn-images-1.medium.com/max/800/1*V6sGeRLDwyIUAHxA35VF6w.png "GraphQL with GoLang"){:class="blog-image"}
</div>

As we can see one job is created under Position’s node(as configured in database
layer) and we get the same response in graphql also with newly created id.

Now let’s test query:

<div markdown="1" class="blog-image-container">
![GraphQL with GoLang](https://cdn-images-1.medium.com/max/800/1*KfGVoAmuO9Ax0in7SuGcSw.png "GraphQL with GoLang"){:class="blog-image"}
</div>

But wait, WHAATT! We got an error. Don’t panic its because graphql don’t
understand how to fetch `CreatedBy` User object from just a `CreatedBy` userId
so we need to implement resolver method for that which currently states that
`panic(“not implemented”)` .

```go
func (r *jobResolver) CreatedBy(ctx context.Context, obj *dal.Job) (dal.User, error) {
	userRepository, err := dal.NewUserFirebaseRepository(r.db.App, r.db.Context)
	if err != nil {
		fmt.Printf("Error Fetching NewTeamFirebaseRepository", err)
		return dal.User{}, err
	}
	user, err := userRepository.GetByID(obj.CreatedBy)
	if err != nil {
		fmt.Printf("Error Fetching user", err)
		return dal.User{}, err
	}
	return user, err
}
```

So now hit the query again:

<div markdown="1" class="blog-image-container">
![GraphQL with GoLang](https://cdn-images-1.medium.com/max/800/1*MsbiYSbVjs09QqBXFotflQ.png "GraphQL with GoLang"){:class="blog-image"}
</div>


And it’s all working.

### Now What?

This article just gives a basic demo of how graphql server can be implemented in
the golang(I know there can be many changes in the code and structure). This
code is on
[Github](https://github.com/ridhamtarpara/go-graphql-example/tree/feat/blog-1).
I will write article soon to add CORS, Validation, Authentication, and
Configuration with Golang gqlgen server. Till then happy coding.

Part-2 is on the way!
