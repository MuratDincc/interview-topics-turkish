# GraphQL

## Giriş

GraphQL, Facebook tarafından geliştirilen modern bir API query language'dır. REST API'lerin sınırlamalarını aşarak, client'ların ihtiyaç duydukları veriyi tam olarak almalarını sağlar.

## GraphQL vs REST

### REST API Sınırlamaları
```http
# REST API - Over-fetching örneği
GET /api/users/123
Response: {
  "id": 123,
  "name": "Ahmet",
  "email": "ahmet@example.com",
  "phone": "+90 555 123 4567",
  "address": "İstanbul, Türkiye",
  "birthDate": "1990-01-01",
  "profilePicture": "https://...",
  "lastLogin": "2024-01-15T10:30:00Z"
}

# Client sadece name ve email istiyor ama tüm veri geliyor
```

### GraphQL Çözümü
```graphql
# GraphQL Query - Sadece ihtiyaç duyulan veri
query GetUserBasicInfo($id: ID!) {
  user(id: $id) {
    name
    email
  }
}

# Response - Sadece istenen veri
{
  "data": {
    "user": {
      "name": "Ahmet",
      "email": "ahmet@example.com"
    }
  }
}
```

## GraphQL Temel Kavramlar

### Schema Definition
```graphql
# Schema definition
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
  profile: Profile
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
  createdAt: DateTime!
}

type Profile {
  bio: String
  avatar: String
  website: String
}

type Query {
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
  post(id: ID!): Post
  posts(authorId: ID): [Post!]!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

input CreateUserInput {
  name: String!
  email: String!
  password: String!
}

input UpdateUserInput {
  name: String
  email: String
  bio: String
}
```

### Resolvers
```csharp
// ASP.NET Core GraphQL resolver örneği
public class UserResolver
{
    private readonly IUserService _userService;
    private readonly IPostService _postService;

    public UserResolver(IUserService userService, IPostService postService)
    {
        _userService = userService;
        _postService = postService;
    }

    public async Task<User> GetUser(string id)
    {
        return await _userService.GetByIdAsync(id);
    }

    public async Task<IEnumerable<Post>> GetUserPosts(User user, int? limit = null)
    {
        var posts = await _postService.GetByAuthorIdAsync(user.Id);
        
        if (limit.HasValue)
            posts = posts.Take(limit.Value);
            
        return posts;
    }

    public async Task<Profile> GetUserProfile(User user)
    {
        return await _userService.GetProfileAsync(user.Id);
    }
}
```

## GraphQL Implementation (.NET)

### HotChocolate Setup
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// GraphQL services
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddType<UserType>()
    .AddType<PostType>()
    .AddType<ProfileType>()
    .AddDataLoader<UserDataLoader>()
    .AddFiltering()
    .AddSorting()
    .AddProjections();

var app = builder.Build();

// GraphQL endpoint
app.MapGraphQL();

app.Run();
```

### Type Definitions
```csharp
public class UserType : ObjectType<User>
{
    protected override void Configure(IObjectTypeDescriptor<User> descriptor)
    {
        descriptor.Field(u => u.Id).Type<NonNullType<IdType>>();
        descriptor.Field(u => u.Name).Type<NonNullType<StringType>>();
        descriptor.Field(u => u.Email).Type<NonNullType<StringType>>();
        
        descriptor.Field("posts")
            .ResolveWith<UserResolver>(r => r.GetUserPosts(default!, default!))
            .UseDbContext<ApplicationDbContext>()
            .UsePaging()
            .UseFiltering()
            .UseSorting();
            
        descriptor.Field("profile")
            .ResolveWith<UserResolver>(r => r.GetUserProfile(default!))
            .UseDbContext<ApplicationDbContext>();
    }
}
```

### Query Implementation
```csharp
public class Query
{
    public async Task<User?> GetUser(string id, [Service] IUserService userService)
    {
        return await userService.GetByIdAsync(id);
    }

    public async Task<IEnumerable<User>> GetUsers(
        int? limit = null, 
        int? offset = null,
        [Service] IUserService userService)
    {
        var users = await userService.GetAllAsync();
        
        if (offset.HasValue)
            users = users.Skip(offset.Value);
            
        if (limit.HasValue)
            users = users.Take(limit.Value);
            
        return users;
    }

    public async Task<Post?> GetPost(string id, [Service] IPostService postService)
    {
        return await postService.GetByIdAsync(id);
    }
}
```

### Mutation Implementation
```csharp
public class Mutation
{
    public async Task<User> CreateUser(
        CreateUserInput input,
        [Service] IUserService userService)
    {
        var user = new User
        {
            Name = input.Name,
            Email = input.Email,
            PasswordHash = HashPassword(input.Password)
        };
        
        return await userService.CreateAsync(user);
    }

    public async Task<User> UpdateUser(
        string id,
        UpdateUserInput input,
        [Service] IUserService userService)
    {
        var user = await userService.GetByIdAsync(id);
        if (user == null)
            throw new UserNotFoundException(id);
            
        if (input.Name != null)
            user.Name = input.Name;
            
        if (input.Email != null)
            user.Email = input.Email;
            
        if (input.Bio != null)
            user.Bio = input.Bio;
            
        return await userService.UpdateAsync(user);
    }
}
```

## DataLoader Pattern

### N+1 Problem Çözümü
```csharp
public class UserDataLoader : BatchDataLoader<string, User>
{
    private readonly IUserService _userService;

    public UserDataLoader(IUserService userService, IBatchScheduler batchScheduler)
        : base(batchScheduler)
    {
        _userService = userService;
    }

    protected override async Task<IReadOnlyDictionary<string, User>> LoadBatchAsync(
        IReadOnlyList<string> keys, CancellationToken cancellationToken)
    {
        var users = await _userService.GetByIdsAsync(keys);
        return users.ToDictionary(u => u.Id);
    }
}

// Resolver'da kullanım
public async Task<IEnumerable<Post>> GetUserPosts(User user, int? limit = null)
{
    var posts = await _postService.GetByAuthorIdAsync(user.Id);
    
    if (limit.HasValue)
        posts = posts.Take(limit.Value);
        
    return posts;
}
```

## GraphQL Subscriptions

### Real-time Updates
```csharp
public class Subscription
{
    [Subscribe]
    [Topic("UserCreated")]
    public User OnUserCreated([EventMessage] User user)
    {
        return user;
    }

    [Subscribe]
    [Topic("PostUpdated")]
    public Post OnPostUpdated([EventMessage] Post post)
    {
        return post;
    }
}

// Publisher
public class UserService
{
    private readonly ITopicEventSender _eventSender;

    public async Task<User> CreateUser(CreateUserInput input)
    {
        var user = new User { /* ... */ };
        await _context.Users.AddAsync(user);
        await _context.SaveChangesAsync();
        
        // Publish event
        await _eventSender.SendAsync("UserCreated", user);
        
        return user;
    }
}
```

## Performance Optimization

### Query Complexity Analysis
```csharp
// Program.cs'de complexity limit
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddComplexityAnalyzer(options =>
    {
        options.MaximumAllowedComplexity = 100;
        options.MaximumAllowedDepth = 10;
    });
```

### Caching
```csharp
// Redis cache integration
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddRedisQueryStorage(options =>
    {
        options.ConnectionMultiplexerFactory = sp =>
            Task.FromResult(ConnectionMultiplexer.Connect("localhost:6379"));
    });
```

## Mülakat Soruları

### Temel Sorular

1. **GraphQL nedir ve REST'e göre avantajları nelerdir?**
   - **Cevap**: Query language, over-fetching/under-fetching'i önler, single endpoint, strong typing, real-time updates.

2. **GraphQL'de N+1 problem nedir?**
   - **Cevap**: Multiple database queries, DataLoader pattern ile çözülür, batching ve caching kullanılır.

3. **GraphQL schema nedir?**
   - **Cevap**: API'nin contract'ı, type definitions, queries, mutations ve subscriptions tanımlar.

4. **Resolver nedir ve nasıl çalışır?**
   - **Cevap**: GraphQL field'larının nasıl resolve edileceğini tanımlayan fonksiyonlar.

5. **GraphQL vs REST performance karşılaştırması nasıldır?**
   - **Cevap**: GraphQL single request, REST multiple requests. Caching ve batching önemli.

### Teknik Sorular

1. **GraphQL'de introspection nedir?**
   - **Cevap**: Schema'yı query edebilme, GraphQL Playground, documentation generation.

2. **GraphQL'de error handling nasıl yapılır?**
   - **Cevap**: Errors array, partial results, proper error codes ve messages.

3. **GraphQL'de authentication nasıl implement edilir?**
   - **Cevap**: HTTP headers, context, directives, custom middleware.

4. **GraphQL'de caching stratejileri nelerdir?**
   - **Cevap**: HTTP caching, application-level caching, Redis integration, query result caching.

5. **GraphQL'de rate limiting nasıl uygulanır?**
   - **Cevap**: Query complexity analysis, depth limiting, field limiting, custom middleware.

## Best Practices

1. **Schema Design**
   - Meaningful type names kullanın
   - Proper nullability tanımlayın
   - Input types kullanın
   - Documentation ekleyin

2. **Performance**
   - DataLoader pattern kullanın
   - Query complexity limitleri ayarlayın
   - Caching implement edin
   - Pagination kullanın

3. **Security**
   - Authentication implement edin
   - Authorization kontrol edin
   - Input validation yapın
   - Rate limiting uygulayın

4. **Error Handling**
   - Consistent error formatları kullanın
   - Proper error codes verin
   - User-friendly messages yazın
   - Logging yapın

## Kaynaklar

- [GraphQL Documentation](https://graphql.org/)
- [HotChocolate Documentation](https://chillicream.com/docs/hotchocolate/)
- [GraphQL Best Practices](https://graphql.org/learn/best-practices/)
- [GraphQL Security](https://graphql.org/learn/authorization/)
- [GraphQL Performance](https://graphql.org/learn/thinking-in-graphs/) 