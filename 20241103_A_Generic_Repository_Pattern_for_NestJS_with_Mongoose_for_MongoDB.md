In modern web development, managing data effectively is crucial for building scalable applications. The code snippet we are exploring represents a Generic Repository class that provides a flexible and reusable interface for interacting with MongoDB using Mongoose in a NestJS environment. This article will break down the code, explain its components, and provide use cases for various methods, especially aimed at newcomers to the field.

## Full Code Example

Here is the complete code for the `GenericRepository` class:

```typescript
import { ConflictException, Logger, NotFoundException } from "@nestjs/common";
import { ObjectId } from "mongodb";
import {
  Document,
  FilterQuery,
  FlattenMaps,
  Model,
  QueryOptions,
  SaveOptions,
  UpdateQuery,
  UpdateWithAggregationPipeline,
} from "mongoose";

export class GenericRepository<T extends Document> {
  private readonly internalLogger: Logger;
  private readonly internalModel: Model<T>;

  constructor(model: Model<T>, logger?: Logger) {
    this.internalModel = model;
    this.internalLogger = logger || new Logger(this.constructor.name);
  }

  async create(doc: Partial<T>, saveOptions: SaveOptions = {}): Promise<T> {
    try {
      const createdEntity = new this.internalModel(doc);
      const savedResult = await createdEntity.save(saveOptions);
      return savedResult;
    } catch (error) {
      if (error?.name === "MongoServerError" && error?.code === 11000) {
        this.internalLogger.error("Duplicate key error while creating:", error);
        throw new ConflictException("Document already exists with provided inputs");
      }
      throw error;
    }
  }

  async getAll(
    filter: FilterQuery<T> = {},
    options: QueryOptions = {},
  ): Promise<FlattenMaps<T>[]> {
    try {
      if (!options.sort) {
        options.sort = { createdAt: -1 };
      }
      const result = await this.internalModel
        .find(filter, null, options)
        .lean()
        .exec();
      return result;
    } catch (error) {
      this.internalLogger.error("Error finding entities:", error);
      return [];
    }
  }

  async getOneWhere(
    filter: FilterQuery<T>,
    options: QueryOptions = {},
  ): Promise<T | null> {
    try {
      const result = await this.internalModel
        .findOne(filter, null, options)
        .exec();
      return result;
    } catch (error) {
      this.internalLogger.error("Error finding entity by ID:", error);
      return null;
    }
  }

  async getOneById(id: string, options: QueryOptions = {}): Promise<T | null> {
    try {
      const result = await this.internalModel
        .findOne({ _id: id }, null, options)
        .exec();
      return result;
    } catch (error) {
      this.internalLogger.error("Error finding entity by ID:", error);
      return null;
    }
  }

  async updateOneById(
    documentId: string,
    updated: UpdateWithAggregationPipeline | UpdateQuery<T>,
    options: QueryOptions = {},
  ): Promise<T> {
    try {
      const result = await this.internalModel
        .findOneAndUpdate(
          { _id: documentId },
          { ...updated, updatedAt: new Date() },
          { ...options, new: true },
        )
        .exec();

      if (!result) {
        throw new NotFoundException("Document not found with provided ID");
      }
      return result;
    } catch (error) {
      if (error?.name === "MongoServerError" && error?.code === 11000) {
        this.internalLogger.error("Duplicate key error while updating:", error);
        throw new ConflictException("Document already exists with provided inputs");
      }
      this.internalLogger.error("Error updating one entity:", error);
      throw error;
    }
  }

  async removeOneById(id: string): Promise<boolean> {
    try {
      const { acknowledged } = await this.internalModel
        .deleteOne({ _id: id })
        .exec();
      return acknowledged;
    } catch (error) {
      this.internalLogger.error("Error removing entities:", error);
      throw error;
    }
  }

  async count(filter: FilterQuery<T> = {}): Promise<number> {
    try {
      const count = await this.internalModel.countDocuments(filter).exec();
      return count;
    } catch (error) {
      this.internalLogger.error("Error counting documents:", error);
      throw error;
    }
  }

  async validateObjectIds(listOfIds: string[] = []): Promise<boolean> {
    try {
      if (!Array.isArray(listOfIds) || !listOfIds?.length) {
        return false;
      }
      const objectIdStrings = listOfIds.map(String);
      const objectIds = objectIdStrings.map((id) => new ObjectId(id));

      const result = await this.internalModel
        .find({ _id: { $in: objectIds } })
        .select("_id")
        .lean()
        .exec();

      return listOfIds.length === result?.length;
    } catch (error) {
      this.internalLogger.error("Error during validation:", error);
      return false;
    }
  }
}
```

## Code Breakdown

### Imports

The code begins with several imports:

```typescript
import { ConflictException, Logger, NotFoundException } from "@nestjs/common";
import { ObjectId } from "mongodb";
import {
  Document,
  FilterQuery,
  FlattenMaps,
  Model,
  QueryOptions,
  SaveOptions,
  UpdateQuery,
  UpdateWithAggregationPipeline,
} from "mongoose";
```

- **NestJS Exceptions**: `ConflictException` and `NotFoundException` are exceptions that can be thrown to handle errors gracefully in a NestJS application.
- **ObjectId**: This is a MongoDB data type used to represent the unique identifier for documents.
- **Mongoose Types**: Various types from Mongoose are imported to help define the structure of the repository.

### Class Definition

The `GenericRepository` class is defined with a generic type parameter `<T extends Document>`, meaning it can work with any Mongoose document.

```typescript
export class GenericRepository<T extends Document> {
  private readonly internalLogger: Logger;
  private readonly internalModel: Model<T>;
```

- **internalLogger**: An instance of `Logger` used for logging errors and information.
- **internalModel**: A Mongoose model representing the collection the repository will interact with.

### Constructor

The constructor initializes the model and logger:

```typescript
constructor(model: Model<T>, logger?: Logger) {
  this.internalModel = model;
  this.internalLogger = logger || new Logger(this.constructor.name);
}
```

- The model is passed in as an argument, allowing the repository to interact with a specific MongoDB collection.
- If a logger is not provided, a default logger using the class name is created.

### Methods

Now let’s go through the methods defined in the GenericRepository class, explaining each and providing use cases.

**1. Create:**

```typescript
async create(doc: Partial<T>, saveOptions: SaveOptions = {}): Promise<T> {
  try {
    const createdEntity = new this.internalModel(doc);
    const savedResult = await createdEntity.save(saveOptions);
    return savedResult;
  } catch (error) {
    if (error?.name === "MongoServerError" && error?.code === 11000) {
      this.internalLogger.error("Duplicate key error while creating:", error);
      throw new ConflictException("Document already exists with provided inputs");
    }
    throw error;
  }
}
```

- Purpose: Creates a new document in the database.
- Parameters:
  doc: The data for the new document.
  saveOptions: Options for saving (optional).
- Use Case: This method can be used when adding a new user to a user collection. If a user with the same email exists, it throws a ConflictException.

**2. GetAll**

```typescript
async getAll(filter: FilterQuery<T> = {}, options: QueryOptions = {}): Promise<FlattenMaps<T>[]> {
  try {
    if (!options.sort) {
      options.sort = { createdAt: -1 };
    }
    const result = await this.internalModel.find(filter, null, options).lean().exec();
    return result;
  } catch (error) {
    this.internalLogger.error("Error finding entities:", error);
    return [];
  }
}
```

- Purpose: Retrieves all documents matching the filter.
- Parameters:
  filter: Criteria to filter documents (optional).
  options: Query options like sorting (optional).
- Use Case: Use this method to fetch all products in an e-commerce application, sorted by the most recently added.

**3. GetOneWhere**

```typescript
async getOneWhere(filter: FilterQuery<T>, options: QueryOptions = {}): Promise<T | null> {
  try {
    const result = await this.internalModel.findOne(filter, null, options).exec();
    return result;
  } catch (error) {
    this.internalLogger.error("Error finding entity by ID:", error);
    return null;
  }
}
```

- Purpose: Retrieves a single document based on the provided filter.
- Use Case: Fetch a specific user by their username to display their profile.

**4. GetOneById**

```typescript
async getOneById(id: string, options: QueryOptions = {}): Promise<T | null> {
  try {
    const result = await this.internalModel.findOne({ _id: id }, null, options).exec();
    return result;
  } catch (error) {
    this.internalLogger.error("Error finding entity by ID:", error);
    return null;
  }
}
```

- Purpose: Fetches a document by its unique identifier.
- Use Case: Retrieve a specific order from an orders collection using its ID.

**5. UpdateOneById**

```typescript
async updateOneById(documentId: string, updated: UpdateWithAggregationPipeline | UpdateQuery<T>, options: QueryOptions = {}): Promise<T> {
  try {
    const result = await this.internalModel.findOneAndUpdate(
      { _id: documentId },
      { ...updated, updatedAt: new Date() },
      { ...options, new: true },
    ).exec();

    if (!result) {
      throw new NotFoundException("Document not found with provided ID");
    }
    return result;
  } catch (error) {
    if (error?.name === "MongoServerError" && error?.code === 11000) {
      this.internalLogger.error("Duplicate key error while updating:", error);
      throw new ConflictException("Document already exists with provided inputs");
    }
    this.internalLogger.error("Error updating one entity:", error);
    throw error;
  }
}
```

- Purpose: Updates a document by its ID.
- Parameters:
  documentId: The ID of the document to update.
  updated: The new data to update.
- Use Case: Modify user information, like updating an email address or password.

**6. RemoveOneById**

```typescript
async removeOneById(id: string): Promise<boolean> {
  try {
    const { acknowledged } = await this.internalModel.deleteOne({ _id: id }).exec();
    return acknowledged;
  } catch (error) {
    this.internalLogger.error("Error removing entities:", error);
    throw error;
  }
}
```

- Purpose: Deletes a document from the database by ID.
- Use Case: Remove a user from the database when they request account deletion.

**7. Count**

```typescript
async count(filter: FilterQuery<T> = {}): Promise<number> {
  try {
    const count = await this.internalModel.countDocuments(filter).exec();
    return count;
  } catch (error) {
    this.internalLogger.error("Error counting documents:", error);
    throw error;
  }
}
```

- Purpose: Counts documents matching the filter criteria.
- Use Case: Determine the number of active users in an application.


**8. ValidateObjectIds**

```typescript
async validateObjectIds(listOfIds: string[] = []): Promise<boolean> {
  try {
    if (!Array.isArray(listOfIds) || !listOfIds?.length) {
      return false;
    }
    const objectIdStrings = listOfIds.map(String);
    const objectIds = objectIdStrings.map((id) => new ObjectId(id));

    const result = await this.internalModel
      .find({ _id: { $in: objectIds } })
      .select("_id")
      .lean()
      .exec();

    return listOfIds.length === result?.length;
  } catch (error) {
    this.internalLogger.error("Error during validation:", error);
    return false;
  }
}
```

- Purpose: Validates a list of Object IDs to ensure they exist in the database.
- Use Case: Before performing bulk operations, ensure all provided IDs are valid.


--- 

The `GenericRepository` class is a powerful pattern for managing data access in a NestJS application using Mongoose. It encapsulates common database operations and promotes code reusability, making it easier for developers to interact with MongoDB collections. By utilizing this pattern, you can enhance the maintainability and scalability of your applications, allowing you to focus more on business logic rather than repetitive database code.