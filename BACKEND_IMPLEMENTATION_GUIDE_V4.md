# Backend Implementation Guide (v4)

**Document Version:** 1.0
**Date:** 2025-09-08 23:24:16
**Author:** @copilot
**Audience:** Backend Engineering Team, @khanhtt

**Objective:** To provide a definitive, copy-pasteable blueprint for creating any backend microservice (e.g., Catalog, Order, Customer, Promotion) using the Hexagonal Architecture on an AWS Serverless stack.

---

## 1. The Canonical Service Structure

Every microservice in the Bookee platform **must** adhere to this directory structure. This consistency is non-negotiable and is our primary tool for ensuring long-term maintainability, testability, and developer onboarding speed.

``
/catalog-service
|
├── /src
│   ├── /domain
│   │   ├── book.js               # The Book entity (Aggregate Root). Contains pure business logic and rules.
│   │   └── catalog_service.js      # The Application Service. Orchestrates domain logic, but contains no logic itself.
│   │
│   ├── /ports
│   │   ├── book_repository.js      # The interface (contract) for storing and retrieving book data.
│   │   └── isbndb_gateway.js       # The interface for fetching external book data from ISBNdb.
│   │
│   ├── /adapters
│   │   ├── /driving                # Adapters that trigger our application. The "entry points".
│   │   │   └── graphql_resolver.js # The Lambda handler for AppSync events.
│   │   │
│   │   └── /driven                 # Adapters used by our application to talk to the outside world.
│   │       ├── dynamodb_repo.js    # The implementation of BookRepository for DynamoDB.
│   │       └── isbndb_http_gw.js   # The implementation of ISBNdbGateway using an HTTP client.
│   │
│   └── /config
│       └── dependency_injector.js  # Wires up all the dependencies. The heart of the hexagonal setup.
│
├── /tests
│   ├── /domain
│   │   └── catalog_service.test.js # Unit tests for the service (using mock adapters).
│   └── /adapters
│       └── dynamodb_repo.test.js   # Integration tests for the DynamoDB adapter.
│
└── serverless.yml                  # Serverless Framework configuration for this specific service.
```

---

## 2. The Code: A Deep Dive into Each Layer

This section provides a complete, working implementation of the `getBookByISBN` flow for the `catalog-service`.

### Layer 1: The Domain (The "What" - The Core Logic)

This layer is pure, dependency-free business logic. It should be written in plain TypeScript/JavaScript and must not contain any reference to AWS, databases, or external frameworks.

**File: `src/domain/book.js`**
```javascript
// The Book entity represents a book in our system. It contains business rules.
export class Book {
  constructor({ id, title, price, stock, isbndbSyncTimestamp, authors, publisher, publishedDate, pageCount, coverImageUrl, avgRating = 0 }) {
    if (!id) throw new Error("Book ID (ISBN) is required.");
    this.id = id; // ISBN-13
    this.title = title;
    this.price = price; // Our internal price
    this.stock = stock; // Our internal stock
    this.isbndbSyncTimestamp = isbndbSyncTimestamp;
    this.authors = authors || [];
    this.publisher = publisher;
    this.publishedDate = publishedDate;
    this.pageCount = pageCount;
    this.coverImageUrl = coverImageUrl;
    this.avgRating = avgRating;
  }

  // Business Rule: Defines what "fresh" means for our cache.
  isCacheFresh(now = new Date()) {
    if (!this.isbndbSyncTimestamp) return false;
    const oneDay = 24 * 60 * 60 * 1000;
    return (now - new Date(this.isbndbSyncTimestamp)) < oneDay;
  }

  // Factory Method: Creates a Book entity from external data.
  static fromExternalData(externalData) {
    return new Book({
      id: externalData.isbn13,
      title: externalData.title,
      authors: externalData.authors,
      publisher: externalData.publisher,
      publishedDate: externalData.publish_date,
      pageCount: externalData.pages,
      coverImageUrl: externalData.image,
      // Our internal data (price, stock) will be enriched separately.
      price: 0,
      stock: 0,
    });
  }

  // Data Transfer Object: Returns a plain object representation for APIs.
  toDTO() {
    return {
      id: this.id,
      title: this.title,
      price: this.price,
      stock: this.stock,
      authors: this.authors,
      publisher: this.publisher,
      publishedDate: this.publishedDate,
      pageCount: this.pageCount,
      coverImageUrl: this.coverImageUrl,
      avgRating: this.avgRating,
    };
  }
}
```

**File: `src/domain/catalog_service.js`**
```javascript
// The Application Service orchestrates the use case. It contains no business logic itself.
export class CatalogService {
  constructor({ bookRepository, isbndbGateway }) {
    this.bookRepository = bookRepository;
    this.isbndbGateway = isbndbGateway;
  }

  async getBookByISBN(isbn) {
    // 1. Use a port to check our internal repository (cache) first.
    const cachedBook = await this.bookRepository.findBookByISBN(isbn);

    // 2. Apply domain logic from the entity.
    if (cachedBook && cachedBook.isCacheFresh()) {
      return cachedBook;
    }

    // 3. Cache miss or stale data: use a port to fetch from the external API.
    const externalData = await this.isbndbGateway.fetchByISBN(isbn);
    if (!externalData) {
      throw new Error(`Book with ISBN ${isbn} not found.`);
    }

    // 4. Use the domain entity's factory to create a new Book instance.
    const book = Book.fromExternalData(externalData);

    // 5. Enrich the new instance with our existing internal data (if any).
    book.price = cachedBook?.price || 0;
    book.stock = cachedBook?.stock || 0;
    book.avgRating = cachedBook?.avgRating || 0;

    // 6. Asynchronously update our cache via a port. Fire-and-forget.
    this.bookRepository.save(book).catch(err => {
      // In a real system, this would go to a proper logger.
      console.error("Failed to save book to cache", { isbn: book.id, error: err.message });
    });

    return book;
  }
}
```

### Layer 2: The Ports (The "Contracts")

These are abstract classes or interfaces that define the contract between the domain and the outside world.

**File: `src/ports/book_repository.js`**
```javascript
export class BookRepository {
  async findBookByISBN(isbn) { throw new Error("BookRepository.findBookByISBN not implemented"); }
  async save(book) { throw new Error("BookRepository.save not implemented"); }
}
```

**File: `src/ports/isbndb_gateway.js`**
```javascript
export class ISBNdbGateway {
  async fetchByISBN(isbn) { throw new Error("ISBNdbGateway.fetchByISBN not implemented"); }
}
```

### Layer 3: The Driven Adapters (The "How-To")

These adapters implement the ports and contain all infrastructure-specific code.

**File: `src/adapters/driven/dynamodb_repo.js`**
```javascript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand, PutCommand } from '@aws-sdk/lib-dynamodb';
import { BookRepository } from '../../ports/book_repository';
import { Book } from '../../domain/book';

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

export class DynamoDBBookRepository extends BookRepository {
  constructor() {
    super();
    this.tableName = process.env.TABLE_NAME;
  }

  async findBookByISBN(isbn) {
    const command = new GetCommand({
      TableName: this.tableName,
      Key: { PK: `BOOK#${isbn}`, SK: 'METADATA' },
    });
    const { Item } = await docClient.send(command);
    return Item ? new Book(Item) : null;
  }

  async save(book) {
    const item = {
      PK: `BOOK#${book.id}`,
      SK: 'METADATA',
      ...book.toDTO(), // Use the DTO to get all serializable fields
      isbndbSyncTimestamp: new Date().toISOString(),
      // Construct GSI attributes based on the schema design
      GSI2PK: 'BOOK#METADATA',
      GSI2SK: book.avgRating || 0,
    };
    const command = new PutCommand({
      TableName: this.tableName,
      Item: item,
    });
    await docClient.send(command);
  }
}
```

**File: `src/adapters/driven/isbndb_http_gw.js`**
```javascript
import { ISBNdbGateway } from '../../ports/isbndb_gateway';
import fetch from 'node-fetch';

export class ISBNdbHttpGateway extends ISBNdbGateway {
  constructor(apiKey) {
    super();
    this.apiKey = apiKey;
    this.baseUrl = 'https://api.isbndb.com';
  }

  async fetchByISBN(isbn) {
    const headers = { 'Authorization': this.apiKey, 'Content-Type': 'application/json' };
    try {
      const response = await fetch(`${this.baseUrl}/book/${isbn}`, { headers });
      if (response.status === 404) return null;
      if (!response.ok) {
        throw new Error(`ISBNdb API failed with status ${response.status}`);
      }
      const { book } = await response.json();
      return book;
    } catch (error) {
      console.error("ISBNdb Gateway Error:", error);
      // Depending on policy, you might want to re-throw or return null
      throw error;
    }
  }
}
```

### Layer 4: The Driving Adapter & Configuration (The "Entry Point")

This is the outermost layer that connects the application to the AWS infrastructure.

**File: `src/config/dependency_injector.js`**
```javascript
// This file performs Dependency Injection. It instantiates the adapters
// and injects them into the domain service, creating a fully configured application instance.
import { CatalogService } from '../domain/catalog_service';
import { DynamoDBBookRepository } from '../adapters/driven/dynamodb_repo';
import { ISBNdbHttpGateway } from '../adapters/driven/isbndb_http_gw';

// 1. Instantiate Driven Adapters with their specific configurations.
const bookRepository = new DynamoDBBookRepository();
const isbndbGateway = new ISBNdbHttpGateway(process.env.ISBNDB_API_KEY);

// 2. Inject the concrete adapter instances into the domain service.
// The domain service does not know what kind of repository or gateway it's getting,
// only that it fulfills the contract defined by the port.
export const catalogService = new CatalogService({ bookRepository, isbndbGateway });
```

**File: `src/adapters/driving/graphql_resolver.js`**
```javascript
// This is the Lambda Handler function. Its only responsibility is to:
// 1. Adapt the incoming AppSync event into a format the domain service understands.
// 2. Call the domain service.
// 3. Adapt the result from the domain service back into a format AppSync understands.
import { catalogService } from '../../config/dependency_injector';

export async function getBookByISBNResolver(event) {
  const { isbn } = event.arguments;
  // It's best practice to extract a correlation ID for tracing across services.
  const correlationId = event.request?.headers?.['x-correlation-id'] || `lambda-${process.env.AWS_LAMBDA_LOG_STREAM_NAME}`;

  try {
    // Log entry with context
    console.log(JSON.stringify({ correlationId, type: 'entry', function: 'getBookByISBNResolver', isbn }));
    
    // Call the fully configured application service.
    const book = await catalogService.getBookByISBN(isbn);

    // Adapt the domain entity back to a plain DTO for the GraphQL response.
    return book.toDTO();

  } catch (error) {
    // Log the error with context before re-throwing.
    console.error(JSON.stringify({ correlationId, type: 'error', function: 'getBookByISBNResolver', error: error.message, stack: error.stack }));
    
    // Let AppSync handle formatting the error for the GraphQL client.
    throw error;
  }
}
```