# Healthy Food Delivery Service API(NutriGo)

A RESTful backend API for a healthy food delivery platform built with **Spring Boot 4.1**, **MongoDB**, and **AWS S3**. It provides secure authentication, food catalog management with image uploads, and order tracking with role-based access control.

---

##  Tech Stack

| Layer | Technology |
|---|---|
| Framework | Spring Boot 4.1 |
| Language | Java 21 |
| Database | MongoDB |
| Security | Spring Security + JWT (JJWT 0.11.5) |
| File Storage | AWS S3 (SDK v2) |
| Validation | Spring Validation (Jakarta) |
| Boilerplate | Lombok 1.18.46 |
| Build Tool | Maven (Maven Wrapper included) |

---

## Project Structure

```
src/main/java/com/healthyfood/hfapi/
├── HfapiApplication.java          # Application entry point
├── config/
│   ├── AWSConfig.java             # AWS S3 client bean
│   ├── ApplicationConfig.java     # Auth provider & password encoder
│   ├── JwtAuthenticationFilter.java  # JWT request filter
│   ├── JwtService.java            # JWT generation & validation
│   └── SecurityConfig.java        # HTTP security rules
├── controller/
│   ├── AuthController.java        # /api/auth endpoints
│   ├── FoodController.java        # /api/foods endpoints
│   └── OrderController.java       # /api/orders endpoints
├── dto/                           # Request/Response DTOs
├── exception/                     # Custom exceptions
├── model/
│   ├── FoodModel.java             # Food document
│   ├── OrderModel.java            # Order document
│   ├── OrderItem.java             # Embedded order item
│   └── UserModel.java             # User document
├── repo/                          # MongoDB repositories
└── service/
    ├── AuthService.java           # Registration & login logic
    ├── FoodService.java           # Food service interface
    ├── FoodServiceImpl.java       # Food CRUD + S3 upload logic
    ├── OrderService.java          # Order service interface
    └── OrderServiceImpl.java      # Order management logic
```

---

## Configuration

Create an `application.properties` (or `application.yml`) under `src/main/resources/` with the following properties:

```properties
# MongoDB
spring.data.mongodb.uri=mongodb://localhost:27017/healthyfood

# JWT
jwt.secret=your-256-bit-secret
jwt.expiration=86400000

# AWS S3
aws.access.key=YOUR_AWS_ACCESS_KEY
aws.secret.key=YOUR_AWS_SECRET_KEY
aws.region=us-east-1
aws.s3.bucket-name=your-s3-bucket-name
```

> **Note:** The AWS S3 client bean is conditionally created — if `aws.access.key` and `aws.secret.key` are not provided, the bean won't be registered and file upload endpoints will fail at runtime.

---

##  Authentication

The API uses **stateless JWT-based authentication**. Include the token in the `Authorization` header for protected endpoints.

```
Authorization: Bearer <your_jwt_token>
```

### Auth Endpoints

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/auth/register` | Register a new user | ❌ |
| `POST` | `/api/auth/login` | Login and receive JWT | ❌ |

#### Register Request Body
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepassword",
  "role": "USER"
}
```

#### Login Request Body
```json
{
  "email": "john@example.com",
  "password": "securepassword"
}
```

#### Auth Response
```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9..."
}
```

---

## Food Endpoints

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `GET` | `/api/foods` | Get all food items | ❌ |
| `GET` | `/api/foods/{id}` | Get food item by ID | ❌ |
| `GET` | `/api/foods/paginated` | Get foods with pagination, search & filter | ❌ |
| `POST` | `/api/foods` | Add a new food item (with image) | ✅ |
| `PUT` | `/api/foods/{id}` | Update a food item (with optional new image) | ✅ |
| `DELETE` | `/api/foods/{id}` | Delete a food item | ✅ |

### Paginated Foods Query Params

| Param | Type | Default | Description |
|---|---|---|---|
| `search` | `String` | — | Filter by name (case-insensitive) |
| `maxPrice` | `Double` | — | Filter by maximum price |
| `page` | `int` | `0` | Page number |
| `size` | `int` | `5` | Items per page |
| `sortBy` | `String` | `id` | Field to sort by |

### Add / Update Food (multipart/form-data)

| Part | Type | Description |
|---|---|---|
| `food` | JSON string | Food details (see below) |
| `file` | File | Food image (required on create, optional on update) |

#### Food JSON Schema
```json
{
  "name": "Quinoa Salad",
  "description": "Fresh quinoa with vegetables",
  "price": 12.99,
  "category": "Salads",
  "calories": 350,
  "protein": 12.5,
  "carbs": 45.0,
  "fat": 8.0,
  "dietaryPreferences": ["vegan", "gluten-free"],
  "allergens": ["nuts"],
  "available": true,
  "preparationTimeMinutes": 15,
  "portionSize": "300g"
}
```

---

## Order Endpoints

| Method | Endpoint | Description | Auth Required | Role |
|---|---|---|---|---|
| `POST` | `/api/orders` | Place a new order | ✅ | USER / ADMIN |
| `GET` | `/api/orders/my-orders` | Get authenticated user's orders | ✅ | USER / ADMIN |
| `GET` | `/api/orders` | Get all orders | ✅ | **ADMIN only** |
| `PATCH` | `/api/orders/{id}/status` | Update order status | ✅ | **ADMIN only** |

#### Create Order Request Body
```json
{
  "items": [
    { "foodId": "abc123", "quantity": 2 },
    { "foodId": "def456", "quantity": 1 }
  ]
}
```

#### Update Order Status Query Param
```
PATCH /api/orders/{id}/status?status=DELIVERED
```

---

##  Security Rules

| Endpoint Pattern | Access |
|---|---|
| `/api/auth/**` | Public |
| `/api/foods/**` | Public (read); JWT required for write operations |
| `/api/orders/**` | JWT required; admin-only for `GET /api/orders` and status updates |

---

##  Data Models

### FoodModel (`foods` collection)
| Field | Type | Description |
|---|---|---|
| `id` | String | MongoDB ObjectId |
| `name` | String | Food name |
| `description` | String | Food description |
| `price` | double | Price in currency units |
| `category` | String | Food category |
| `imageUrl` | String | Public S3 URL |
| `calories` | double | Caloric value |
| `protein` | double | Protein (grams) |
| `carbs` | double | Carbohydrates (grams) |
| `fat` | double | Fat (grams) |
| `dietaryPreferences` | List<String> | e.g., vegan, keto |
| `allergens` | List<String> | e.g., nuts, dairy |
| `isAvailable` | boolean | Availability flag |
| `preparationTimeMinutes` | int | Prep time in minutes |
| `portionSize` | String | e.g., "300g" |

### OrderModel (`orders` collection)
| Field | Type | Description |
|---|---|---|
| `id` | String | MongoDB ObjectId |
| `userEmail` | String | Linked user's email |
| `items` | List<OrderItem> | Ordered items |
| `totalPrice` | Double | Computed total |
| `status` | String | e.g., PENDING, DELIVERED |
| `orderDate` | String | ISO date string |

### UserModel (`users` collection)
| Field | Type | Description |
|---|---|---|
| `id` | String | MongoDB ObjectId |
| `name` | String | Full name |
| `email` | String | Unique email (used as username) |
| `password` | String | Bcrypt-hashed password |
| `role` | String | `USER` or `ADMIN` |

---

##  Running the Application

### Prerequisites

- Java 21+
- MongoDB running locally (or provide a cloud URI)
- AWS S3 bucket (for image uploads)

### Steps

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd healthy_food_delivery_service
   ```

2. **Configure properties**
   Create `src/main/resources/application.properties` and fill in the values from the [Configuration](#️-configuration) section.

3. **Run using Maven Wrapper**
   ```bash
   # Windows
   .\mvnw.cmd spring-boot:run

   # Linux / macOS
   ./mvnw spring-boot:run
   ```

4. The API will be available at: `http://localhost:8080`

### Running Tests
```bash
.\mvnw.cmd test
```

### Building a JAR
```bash
.\mvnw.cmd clean package
java -jar target/hfapi-0.0.1-SNAPSHOT.jar
```

---

##  CORS

The API allows cross-origin requests from `http://localhost:5173` (Vite dev server default). Update the `@CrossOrigin` annotation on each controller for production deployments.

---

##  Image Storage (AWS S3)

Food images are uploaded directly to S3. On upload:
- A UUID-prefixed filename is generated to prevent collisions.
- The public URL format is: `https://<bucket-name>.s3.amazonaws.com/<filename>`

On food deletion or image replacement, the old S3 object is automatically removed.

