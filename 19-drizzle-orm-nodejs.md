# Drizzle ORM with PostgreSQL in Node.js

*Complete Guide to Drizzle ORM Operations*

## Table of Contents
1. [Setup and Installation](#setup-and-installation)
2. [Basic Schema Definition](#basic-schema-definition)
3. [Database Connection](#database-connection)
4. [Basic CRUD Operations](#basic-crud-operations)
5. [Intermediate Operations](#intermediate-operations)
6. [Indexing and Performance](#indexing-and-performance)
7. [Advanced Queries](#advanced-queries)
8. [Migrations](#migrations)
9. [Relationships and Joins](#relationships-and-joins)
10. [Best Practices](#best-practices)

## Setup and Installation

### Package Installation

```bash
# Initialize Node.js project
npm init -y

# Install Drizzle ORM and PostgreSQL driver
npm install drizzle-orm pg
npm install -D drizzle-kit @types/pg

# Install additional utilities
npm install dotenv
npm install -D typescript @types/node ts-node
```

### TypeScript Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Environment Configuration

```env
# .env
DATABASE_URL=postgresql://username:password@localhost:5432/database_name
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=your_password
DB_NAME=drizzle_demo
NODE_ENV=development
```

### Drizzle Configuration

```typescript
// drizzle.config.ts
import type { Config } from 'drizzle-kit';
import * as dotenv from 'dotenv';

dotenv.config();

export default {
  schema: './src/schema/*',
  out: './drizzle',
  driver: 'pg',
  dbCredentials: {
    connectionString: process.env.DATABASE_URL!,
  },
  verbose: true,
  strict: true,
} satisfies Config;
```

## Basic Schema Definition

### User Schema

```typescript
// src/schema/users.ts
import {
  pgTable,
  serial,
  varchar,
  text,
  timestamp,
  boolean,
  integer,
  decimal,
  uuid,
  index,
  uniqueIndex,
} from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

export const users = pgTable(
  'users',
  {
    id: serial('id').primaryKey(),
    uuid: uuid('uuid').defaultRandom().notNull().unique(),
    username: varchar('username', { length: 50 }).notNull().unique(),
    email: varchar('email', { length: 100 }).notNull().unique(),
    firstName: varchar('first_name', { length: 50 }),
    lastName: varchar('last_name', { length: 50 }),
    bio: text('bio'),
    isActive: boolean('is_active').default(true),
    isVerified: boolean('is_verified').default(false),
    loginCount: integer('login_count').default(0),
    lastLoginAt: timestamp('last_login_at'),
    createdAt: timestamp('created_at').defaultNow().notNull(),
    updatedAt: timestamp('updated_at').defaultNow().notNull(),
  },
  (table) => {
    return {
      usernameIdx: index('username_idx').on(table.username),
      emailIdx: uniqueIndex('email_idx').on(table.email),
      activeUsersIdx: index('active_users_idx').on(table.isActive).where(sql`${table.isActive} = true`),
      nameIdx: index('name_idx').on(table.firstName, table.lastName),
      createdAtIdx: index('created_at_idx').on(table.createdAt),
    };
  }
);

export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
```

### Posts Schema

```typescript
// src/schema/posts.ts
import {
  pgTable,
  serial,
  varchar,
  text,
  timestamp,
  integer,
  boolean,
  index,
  foreignKey,
} from 'drizzle-orm/pg-core';
import { users } from './users';

export const posts = pgTable(
  'posts',
  {
    id: serial('id').primaryKey(),
    title: varchar('title', { length: 200 }).notNull(),
    slug: varchar('slug', { length: 250 }).notNull().unique(),
    content: text('content').notNull(),
    excerpt: varchar('excerpt', { length: 500 }),
    authorId: integer('author_id').notNull(),
    viewCount: integer('view_count').default(0),
    likeCount: integer('like_count').default(0),
    isPublished: boolean('is_published').default(false),
    publishedAt: timestamp('published_at'),
    createdAt: timestamp('created_at').defaultNow().notNull(),
    updatedAt: timestamp('updated_at').defaultNow().notNull(),
  },
  (table) => {
    return {
      authorFk: foreignKey({
        columns: [table.authorId],
        foreignColumns: [users.id],
        name: 'posts_author_fk'
      }),
      authorIdx: index('posts_author_idx').on(table.authorId),
      slugIdx: index('posts_slug_idx').on(table.slug),
      publishedIdx: index('posts_published_idx').on(table.isPublished, table.publishedAt),
      titleSearchIdx: index('posts_title_search_idx').using('gin', sql`to_tsvector('english', ${table.title})`),
      contentSearchIdx: index('posts_content_search_idx').using('gin', sql`to_tsvector('english', ${table.content})`),
    };
  }
);

export type Post = typeof posts.$inferSelect;
export type NewPost = typeof posts.$inferInsert;
```

### Categories and Tags Schema

```typescript
// src/schema/categories.ts
import {
  pgTable,
  serial,
  varchar,
  text,
  timestamp,
  integer,
  index,
} from 'drizzle-orm/pg-core';

export const categories = pgTable(
  'categories',
  {
    id: serial('id').primaryKey(),
    name: varchar('name', { length: 100 }).notNull().unique(),
    slug: varchar('slug', { length: 120 }).notNull().unique(),
    description: text('description'),
    parentId: integer('parent_id'),
    sortOrder: integer('sort_order').default(0),
    createdAt: timestamp('created_at').defaultNow().notNull(),
    updatedAt: timestamp('updated_at').defaultNow().notNull(),
  },
  (table) => {
    return {
      nameIdx: index('categories_name_idx').on(table.name),
      slugIdx: index('categories_slug_idx').on(table.slug),
      parentIdx: index('categories_parent_idx').on(table.parentId),
      sortIdx: index('categories_sort_idx').on(table.sortOrder),
    };
  }
);

export const tags = pgTable(
  'tags',
  {
    id: serial('id').primaryKey(),
    name: varchar('name', { length: 50 }).notNull().unique(),
    slug: varchar('slug', { length: 60 }).notNull().unique(),
    color: varchar('color', { length: 7 }).default('#000000'),
    usageCount: integer('usage_count').default(0),
    createdAt: timestamp('created_at').defaultNow().notNull(),
  },
  (table) => {
    return {
      nameIdx: index('tags_name_idx').on(table.name),
      usageIdx: index('tags_usage_idx').on(table.usageCount),
    };
  }
);

// Junction table for many-to-many relationship
export const postTags = pgTable(
  'post_tags',
  {
    postId: integer('post_id').notNull(),
    tagId: integer('tag_id').notNull(),
    createdAt: timestamp('created_at').defaultNow().notNull(),
  },
  (table) => {
    return {
      pk: index('post_tags_pk').on(table.postId, table.tagId),
      postIdx: index('post_tags_post_idx').on(table.postId),
      tagIdx: index('post_tags_tag_idx').on(table.tagId),
    };
  }
);

export type Category = typeof categories.$inferSelect;
export type NewCategory = typeof categories.$inferInsert;
export type Tag = typeof tags.$inferSelect;
export type NewTag = typeof tags.$inferInsert;
export type PostTag = typeof postTags.$inferSelect;
export type NewPostTag = typeof postTags.$inferInsert;
```

## Database Connection

### Connection Setup

```typescript
// src/db/connection.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as dotenv from 'dotenv';
import * as schema from '../schema';

dotenv.config();

// Create connection pool
const pool = new Pool({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  max: 20, // Maximum number of clients in the pool
  idleTimeoutMillis: 30000, // Close idle clients after 30 seconds
  connectionTimeoutMillis: 2000, // Return an error after 2 seconds if connection could not be established
});

// Create Drizzle instance
export const db = drizzle(pool, { schema });

// Connection health check
export async function checkConnection(): Promise<boolean> {
  try {
    const client = await pool.connect();
    await client.query('SELECT 1');
    client.release();
    console.log('✅ Database connection successful');
    return true;
  } catch (error) {
    console.error('❌ Database connection failed:', error);
    return false;
  }
}
```

## Migrations

### Migration Setup

```typescript
// src/migrations/migrate.ts
import { migrate } from 'drizzle-orm/node-postgres/migrator';
import { db } from '../db/connection';
import path from 'path';

export async function runMigrations() {
  try {
    console.log('🔄 Running migrations...');
    
    await migrate(db, {
      migrationsFolder: path.join(__dirname, '../../drizzle'),
    });
    
    console.log('✅ Migrations completed successfully');
  } catch (error) {
    console.error('❌ Migration failed:', error);
    throw new Error('Failed to run migrations');
  }
}

// Run migrations if this file is executed directly
if (require.main === module) {
  runMigrations()
    .then(() => {
      console.log('Migration process completed');
      process.exit(0);
    })
    .catch((error) => {
      console.error('Migration process failed:', error);
      process.exit(1);
    });
}
```

### Migration Scripts

```json
// package.json scripts
{
  "scripts": {
    "db:generate": "drizzle-kit generate:pg",
    "db:migrate": "ts-node src/migrations/migrate.ts",
    "db:studio": "drizzle-kit studio",
    "db:drop": "drizzle-kit drop",
    "db:seed": "ts-node src/seeds/seed.ts",
    "db:reset": "npm run db:drop && npm run db:migrate && npm run db:seed"
  }
}
```

### Seed Data

```typescript
// src/seeds/seed.ts
import { db } from '../db/connection';
import { users, posts, categories, tags, postTags } from '../schema';
import { faker } from '@faker-js/faker';

export async function seedDatabase() {
  try {
    console.log('🌱 Seeding database...');
    
    // Clear existing data
    await db.delete(postTags);
    await db.delete(posts);
    await db.delete(tags);
    await db.delete(categories);
    await db.delete(users);
    
    // Seed users
    const userIds: number[] = [];
    for (let i = 0; i < 10; i++) {
      const [user] = await db
        .insert(users)
        .values({
          username: faker.internet.userName(),
          email: faker.internet.email(),
          firstName: faker.person.firstName(),
          lastName: faker.person.lastName(),
          bio: faker.lorem.paragraph(),
          isActive: faker.datatype.boolean(0.9),
          isVerified: faker.datatype.boolean(0.7),
        })
        .returning({ id: users.id });
      
      userIds.push(user.id);
    }
    
    // Seed categories
    const categoryIds: number[] = [];
    const categoryNames = ['Technology', 'Science', 'Health', 'Travel', 'Food'];
    
    for (const name of categoryNames) {
      const [category] = await db
        .insert(categories)
        .values({
          name,
          slug: name.toLowerCase(),
          description: faker.lorem.sentence(),
        })
        .returning({ id: categories.id });
      
      categoryIds.push(category.id);
    }
    
    // Seed tags
    const tagIds: number[] = [];
    const tagNames = [
      'javascript', 'typescript', 'react', 'nodejs', 'postgresql',
      'health', 'fitness', 'nutrition', 'travel', 'adventure',
      'cooking', 'recipes', 'science', 'research', 'technology'
    ];
    
    for (const name of tagNames) {
      const [tag] = await db
        .insert(tags)
        .values({
          name,
          slug: name.toLowerCase(),
          color: faker.internet.color(),
        })
        .returning({ id: tags.id });
      
      tagIds.push(tag.id);
    }
    
    // Seed posts
    const postIds: number[] = [];
    for (let i = 0; i < 50; i++) {
      const title = faker.lorem.sentence();
      const [post] = await db
        .insert(posts)
        .values({
          title,
          slug: title.toLowerCase().replace(/[^a-z0-9]+/g, '-'),
          content: faker.lorem.paragraphs(5),
          excerpt: faker.lorem.paragraph(),
          authorId: faker.helpers.arrayElement(userIds),
          viewCount: faker.number.int({ min: 0, max: 1000 }),
          likeCount: faker.number.int({ min: 0, max: 100 }),
          isPublished: faker.datatype.boolean(0.8),
          publishedAt: faker.datatype.boolean(0.8) ? faker.date.past() : null,
        })
        .returning({ id: posts.id });
      
      postIds.push(post.id);
    }
    
    // Seed post-tag relationships
    for (const postId of postIds) {
      const numTags = faker.number.int({ min: 1, max: 5 });
      const selectedTags = faker.helpers.arrayElements(tagIds, numTags);
      
      for (const tagId of selectedTags) {
        await db
          .insert(postTags)
          .values({
            postId,
            tagId,
          })
          .onConflictDoNothing();
      }
    }
    
    console.log('✅ Database seeded successfully');
    console.log(`Created ${userIds.length} users`);
    console.log(`Created ${categoryIds.length} categories`);
    console.log(`Created ${tagIds.length} tags`);
    console.log(`Created ${postIds.length} posts`);
    
  } catch (error) {
    console.error('❌ Seeding failed:', error);
    throw new Error('Failed to seed database');
  }
}

// Run seeding if this file is executed directly
if (require.main === module) {
  seedDatabase()
    .then(() => {
      console.log('Seeding process completed');
      process.exit(0);
    })
    .catch((error) => {
      console.error('Seeding process failed:', error);
      process.exit(1);
    });
 }
```

## Relationships and Joins

### One-to-Many Relationships

```typescript
// src/services/relationshipService.ts
import { db } from '../db/connection';
import { users, posts, categories, tags, postTags } from '../schema';
import { eq, and, or, sql, desc } from 'drizzle-orm';

export class RelationshipService {
  // Get user with all their posts
  static async getUserWithPosts(userId: number) {
    try {
      const userWithPosts = await db
        .select({
          // User fields
          userId: users.id,
          username: users.username,
          email: users.email,
          firstName: users.firstName,
          lastName: users.lastName,
          
          // Post fields
          postId: posts.id,
          postTitle: posts.title,
          postSlug: posts.slug,
          postExcerpt: posts.excerpt,
          postViewCount: posts.viewCount,
          postLikeCount: posts.likeCount,
          postPublishedAt: posts.publishedAt,
          postIsPublished: posts.isPublished,
        })
        .from(users)
        .leftJoin(posts, eq(users.id, posts.authorId))
        .where(eq(users.id, userId))
        .orderBy(desc(posts.publishedAt));
      
      // Group posts under user
      if (userWithPosts.length === 0) return null;
      
      const user = {
        id: userWithPosts[0].userId,
        username: userWithPosts[0].username,
        email: userWithPosts[0].email,
        firstName: userWithPosts[0].firstName,
        lastName: userWithPosts[0].lastName,
        posts: userWithPosts
          .filter(row => row.postId !== null)
          .map(row => ({
            id: row.postId,
            title: row.postTitle,
            slug: row.postSlug,
            excerpt: row.postExcerpt,
            viewCount: row.postViewCount,
            likeCount: row.postLikeCount,
            publishedAt: row.postPublishedAt,
            isPublished: row.postIsPublished,
          }))
      };
      
      return user;
    } catch (error) {
      console.error('❌ Error fetching user with posts:', error);
      throw new Error('Failed to fetch user with posts');
    }
  }

  // Get post with author and tags
  static async getPostWithAuthorAndTags(postId: number) {
    try {
      const postWithData = await db
        .select({
          // Post fields
          postId: posts.id,
          postTitle: posts.title,
          postSlug: posts.slug,
          postContent: posts.content,
          postExcerpt: posts.excerpt,
          postViewCount: posts.viewCount,
          postLikeCount: posts.likeCount,
          postPublishedAt: posts.publishedAt,
          
          // Author fields
          authorId: users.id,
          authorUsername: users.username,
          authorFirstName: users.firstName,
          authorLastName: users.lastName,
          authorBio: users.bio,
          
          // Tag fields
          tagId: tags.id,
          tagName: tags.name,
          tagSlug: tags.slug,
          tagColor: tags.color,
        })
        .from(posts)
        .innerJoin(users, eq(posts.authorId, users.id))
        .leftJoin(postTags, eq(posts.id, postTags.postId))
        .leftJoin(tags, eq(postTags.tagId, tags.id))
        .where(eq(posts.id, postId));
      
      if (postWithData.length === 0) return null;
      
      const post = {
        id: postWithData[0].postId,
        title: postWithData[0].postTitle,
        slug: postWithData[0].postSlug,
        content: postWithData[0].postContent,
        excerpt: postWithData[0].postExcerpt,
        viewCount: postWithData[0].postViewCount,
        likeCount: postWithData[0].postLikeCount,
        publishedAt: postWithData[0].postPublishedAt,
        
        author: {
          id: postWithData[0].authorId,
          username: postWithData[0].authorUsername,
          firstName: postWithData[0].authorFirstName,
          lastName: postWithData[0].authorLastName,
          bio: postWithData[0].authorBio,
        },
        
        tags: postWithData
          .filter(row => row.tagId !== null)
          .map(row => ({
            id: row.tagId,
            name: row.tagName,
            slug: row.tagSlug,
            color: row.tagColor,
          }))
      };
      
      return post;
    } catch (error) {
      console.error('❌ Error fetching post with author and tags:', error);
      throw new Error('Failed to fetch post with author and tags');
    }
  }

  // Get all posts with their authors and tag counts
  static async getAllPostsWithAuthorsAndTagCounts() {
    try {
      const postsWithData = await db
        .select({
          postId: posts.id,
          postTitle: posts.title,
          postSlug: posts.slug,
          postExcerpt: posts.excerpt,
          postViewCount: posts.viewCount,
          postLikeCount: posts.likeCount,
          postPublishedAt: posts.publishedAt,
          
          authorUsername: users.username,
          authorFirstName: users.firstName,
          authorLastName: users.lastName,
          
          tagCount: sql<number>`COUNT(DISTINCT ${postTags.tagId})`,
        })
        .from(posts)
        .innerJoin(users, eq(posts.authorId, users.id))
        .leftJoin(postTags, eq(posts.id, postTags.postId))
        .where(eq(posts.isPublished, true))
        .groupBy(posts.id, users.id)
        .orderBy(desc(posts.publishedAt));
      
      return postsWithData;
    } catch (error) {
      console.error('❌ Error fetching posts with authors and tag counts:', error);
      throw new Error('Failed to fetch posts with authors and tag counts');
    }
  }
}
```

## Best Practices

### Connection Management

```typescript
// src/utils/connectionManager.ts
import { Pool } from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';

class ConnectionManager {
  private static instance: ConnectionManager;
  private pool: Pool;
  private db: any;

  private constructor() {
    this.pool = new Pool({
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT || '5432'),
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME,
      max: 20,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
    });
    
    this.db = drizzle(this.pool);
  }

  public static getInstance(): ConnectionManager {
    if (!ConnectionManager.instance) {
      ConnectionManager.instance = new ConnectionManager();
    }
    return ConnectionManager.instance;
  }

  public getDb() {
    return this.db;
  }

  public async closeConnections(): Promise<void> {
    await this.pool.end();
  }

  public async getPoolStats() {
    return {
      totalCount: this.pool.totalCount,
      idleCount: this.pool.idleCount,
      waitingCount: this.pool.waitingCount,
    };
  }
}

export const connectionManager = ConnectionManager.getInstance();
export const db = connectionManager.getDb();
```

### Error Handling

```typescript
// src/utils/errorHandler.ts
export class DatabaseError extends Error {
  constructor(
    message: string,
    public code?: string,
    public details?: any
  ) {
    super(message);
    this.name = 'DatabaseError';
  }
}

export function handleDatabaseError(error: any): never {
  console.error('Database error:', error);
  
  // PostgreSQL specific error codes
  switch (error.code) {
    case '23505': // unique_violation
      throw new DatabaseError('Duplicate entry found', 'DUPLICATE_ENTRY', error.detail);
    case '23503': // foreign_key_violation
      throw new DatabaseError('Referenced record not found', 'FOREIGN_KEY_VIOLATION', error.detail);
    case '23502': // not_null_violation
      throw new DatabaseError('Required field is missing', 'NOT_NULL_VIOLATION', error.detail);
    case '42P01': // undefined_table
      throw new DatabaseError('Table does not exist', 'UNDEFINED_TABLE', error.detail);
    case '42703': // undefined_column
      throw new DatabaseError('Column does not exist', 'UNDEFINED_COLUMN', error.detail);
    default:
      throw new DatabaseError('Database operation failed', 'UNKNOWN_ERROR', error);
  }
}

// Wrapper function for database operations
export async function withErrorHandling<T>(
  operation: () => Promise<T>
): Promise<T> {
  try {
    return await operation();
  } catch (error) {
    return handleDatabaseError(error);
  }
}
```

### Validation and Sanitization

```typescript
// src/utils/validation.ts
import { z } from 'zod';

// User validation schemas
export const createUserSchema = z.object({
  username: z.string().min(3).max(50).regex(/^[a-zA-Z0-9_]+$/),
  email: z.string().email().max(100),
  firstName: z.string().min(1).max(50).optional(),
  lastName: z.string().min(1).max(50).optional(),
  bio: z.string().max(1000).optional(),
});

export const updateUserSchema = createUserSchema.partial();

// Post validation schemas
export const createPostSchema = z.object({
  title: z.string().min(1).max(200),
  slug: z.string().min(1).max(250).regex(/^[a-z0-9-]+$/),
  content: z.string().min(1),
  excerpt: z.string().max(500).optional(),
  authorId: z.number().positive(),
  isPublished: z.boolean().default(false),
});

export const updatePostSchema = createPostSchema.partial();

// Validation middleware
export function validateInput<T>(schema: z.ZodSchema<T>) {
  return (data: unknown): T => {
    try {
      return schema.parse(data);
    } catch (error) {
      if (error instanceof z.ZodError) {
        const errorMessages = error.errors.map(err => 
          `${err.path.join('.')}: ${err.message}`
        ).join(', ');
        throw new Error(`Validation failed: ${errorMessages}`);
      }
      throw error;
    }
  };
}
```

### Caching Strategy

```typescript
// src/utils/cache.ts
import NodeCache from 'node-cache';

class CacheManager {
  private cache: NodeCache;
  
  constructor() {
    this.cache = new NodeCache({
      stdTTL: 600, // 10 minutes default TTL
      checkperiod: 120, // Check for expired keys every 2 minutes
    });
  }
  
  set(key: string, value: any, ttl?: number): boolean {
    return this.cache.set(key, value, ttl);
  }
  
  get<T>(key: string): T | undefined {
    return this.cache.get<T>(key);
  }
  
  del(key: string): number {
    return this.cache.del(key);
  }
  
  flush(): void {
    this.cache.flushAll();
  }
  
  getStats() {
    return this.cache.getStats();
  }
  
  // Cache wrapper for database operations
  async getOrSet<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl?: number
  ): Promise<T> {
    const cached = this.get<T>(key);
    if (cached !== undefined) {
      return cached;
    }
    
    const result = await fetcher();
    this.set(key, result, ttl);
    return result;
  }
}

export const cache = new CacheManager();

// Cache keys generator
export const cacheKeys = {
  user: (id: number) => `user:${id}`,
  userPosts: (userId: number) => `user:${userId}:posts`,
  post: (id: number) => `post:${id}`,
  postWithTags: (id: number) => `post:${id}:with-tags`,
  publishedPosts: (page: number, limit: number) => `posts:published:${page}:${limit}`,
  userStats: (id: number) => `user:${id}:stats`,
  postStats: () => 'posts:stats',
};
```

## Example Usage

### Complete Application Example

```typescript
// src/app.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import { checkConnection, closeConnection } from './db/connection';
import { runMigrations } from './migrations/migrate';
import { UserService } from './services/userService';
import { PostService } from './services/postService';
import { validateInput, createUserSchema, createPostSchema } from './utils/validation';
import { withErrorHandling } from './utils/errorHandler';
import { cache, cacheKeys } from './utils/cache';

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(helmet());
app.use(cors());
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per windowMs
});
app.use(limiter);

// Routes

// Health check
app.get('/health', async (req, res) => {
  try {
    const isDbConnected = await checkConnection();
    const cacheStats = cache.getStats();
    
    res.json({
      status: 'ok',
      database: isDbConnected ? 'connected' : 'disconnected',
      cache: cacheStats,
      timestamp: new Date().toISOString(),
    });
  } catch (error) {
    res.status(500).json({
      status: 'error',
      message: 'Health check failed',
    });
  }
});

// User routes
app.post('/users', async (req, res) => {
  try {
    const validatedData = validateInput(createUserSchema)(req.body);
    
    const user = await withErrorHandling(() => 
      UserService.createUser(validatedData)
    );
    
    // Clear related caches
    cache.del('posts:stats');
    
    res.status(201).json({
      success: true,
      data: user,
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: error.message,
    });
  }
});

app.get('/users/:id', async (req, res) => {
  try {
    const userId = parseInt(req.params.id);
    
    const user = await cache.getOrSet(
      cacheKeys.user(userId),
      () => UserService.getUserById(userId),
      300 // 5 minutes cache
    );
    
    if (!user) {
      return res.status(404).json({
        success: false,
        message: 'User not found',
      });
    }
    
    res.json({
      success: true,
      data: user,
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: error.message,
    });
  }
});

app.get('/users', async (req, res) => {
  try {
    const page = parseInt(req.query.page as string) || 1;
    const limit = parseInt(req.query.limit as string) || 10;
    
    const result = await UserService.getAllUsers(page, limit);
    
    res.json({
      success: true,
      data: result,
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: error.message,
    });
  }
});

// Post routes
app.post('/posts', async (req, res) => {
  try {
    const validatedData = validateInput(createPostSchema)(req.body);
    
    const post = await withErrorHandling(() => 
      PostService.createPost(validatedData)
    );
    
    // Clear related caches
    cache.del('posts:stats');
    cache.del(cacheKeys.userPosts(post.authorId));
    
    res.status(201).json({
      success: true,
      data: post,
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: error.message,
    });
  }
});

app.get('/posts/:id', async (req, res) => {
  try {
    const postId = parseInt(req.params.id);
    
    const post = await cache.getOrSet(
      cacheKeys.postWithTags(postId),
      () => PostService.getPostById(postId),
      600 // 10 minutes cache
    );
    
    if (!post) {
      return res.status(404).json({
        success: false,
        message: 'Post not found',
      });
    }
    
    // Increment view count asynchronously
    PostService.incrementViewCount(postId).catch(console.error);
    
    res.json({
      success: true,
      data: post,
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: error.message,
    });
  }
});

app.get('/posts', async (req, res) => {
  try {
    const page = parseInt(req.query.page as string) || 1;
    const limit = parseInt(req.query.limit as string) || 10;
    
    const cacheKey = cacheKeys.publishedPosts(page, limit);
    const result = await cache.getOrSet(
      cacheKey,
      () => PostService.getPublishedPosts(page, limit),
      300 // 5 minutes cache
    );
    
    res.json({
      success: true,
      data: result,
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: error.message,
    });
  }
});

// Search routes
app.get('/search/posts', async (req, res) => {
  try {
    const searchTerm = req.query.q as string;
    
    if (!searchTerm || searchTerm.length < 3) {
      return res.status(400).json({
        success: false,
        message: 'Search term must be at least 3 characters long',
      });
    }
    
    const results = await PostService.searchPosts(searchTerm);
    
    res.json({
      success: true,
      data: results,
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: error.message,
    });
  }
});

// Error handling middleware
app.use((error: Error, req: express.Request, res: express.Response, next: express.NextFunction) => {
  console.error('Unhandled error:', error);
  
  res.status(500).json({
    success: false,
    message: 'Internal server error',
    ...(process.env.NODE_ENV === 'development' && { error: error.message }),
  });
});

// 404 handler
app.use('*', (req, res) => {
  res.status(404).json({
    success: false,
    message: 'Route not found',
  });
});

// Graceful shutdown
process.on('SIGINT', async () => {
  console.log('Received SIGINT, shutting down gracefully...');
  await closeConnection();
  process.exit(0);
});

process.on('SIGTERM', async () => {
  console.log('Received SIGTERM, shutting down gracefully...');
  await closeConnection();
  process.exit(0);
});

// Start server
async function startServer() {
  try {
    // Check database connection
    const isConnected = await checkConnection();
    if (!isConnected) {
      throw new Error('Failed to connect to database');
    }
    
    // Run migrations
    await runMigrations();
    
    // Start server
    app.listen(PORT, () => {
      console.log(`🚀 Server running on port ${PORT}`);
      console.log(`📊 Health check: http://localhost:${PORT}/health`);
    });
  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);
  }
}

startServer();
```

## Summary

This comprehensive Drizzle ORM guide covers:

- **Setup and Configuration**: Complete project setup with TypeScript
- **Schema Definition**: Advanced schema patterns with indexes and relationships
- **Basic Operations**: Full CRUD operations with error handling
- **Intermediate Features**: Transactions, aggregations, and complex queries
- **Performance Optimization**: Indexing strategies and query monitoring
- **Advanced Patterns**: Complex joins, window functions, and recursive queries
- **Best Practices**: Connection management, error handling, validation, and caching
- **Production Ready**: Complete application example with middleware and proper architecture

Drizzle ORM provides excellent TypeScript support, performance, and developer experience for PostgreSQL applications. The type-safe queries and schema-first approach make it ideal for modern Node.js applications.

// Graceful shutdown
export async function closeConnection(): Promise<void> {
  try {
    await pool.end();
    console.log('🔌 Database connection closed');
  } catch (error) {
    console.error('Error closing database connection:', error);
  }
}

// Handle process termination
process.on('SIGINT', async () => {
  await closeConnection();
  process.exit(0);
});

process.on('SIGTERM', async () => {
  await closeConnection();
  process.exit(0);
});
```

## Basic CRUD Operations

### User Operations

```typescript
// src/services/userService.ts
import { eq, and, or, like, ilike, isNull, isNotNull, desc, asc } from 'drizzle-orm';
import { db } from '../db/connection';
import { users, type User, type NewUser } from '../schema/users';

export class UserService {
  // Create a new user
  static async createUser(userData: NewUser): Promise<User> {
    try {
      const [newUser] = await db
        .insert(users)
        .values({
          ...userData,
          createdAt: new Date(),
          updatedAt: new Date(),
        })
        .returning();
      
      console.log('✅ User created:', newUser.username);
      return newUser;
    } catch (error) {
      console.error('❌ Error creating user:', error);
      throw new Error('Failed to create user');
    }
  }

  // Get user by ID
  static async getUserById(id: number): Promise<User | null> {
    try {
      const [user] = await db
        .select()
        .from(users)
        .where(eq(users.id, id))
        .limit(1);
      
      return user || null;
    } catch (error) {
      console.error('❌ Error fetching user by ID:', error);
      throw new Error('Failed to fetch user');
    }
  }

  // Get user by email
  static async getUserByEmail(email: string): Promise<User | null> {
    try {
      const [user] = await db
        .select()
        .from(users)
        .where(eq(users.email, email))
        .limit(1);
      
      return user || null;
    } catch (error) {
      console.error('❌ Error fetching user by email:', error);
      throw new Error('Failed to fetch user');
    }
  }

  // Get all users with pagination
  static async getAllUsers(page: number = 1, limit: number = 10): Promise<{
    users: User[];
    total: number;
    page: number;
    totalPages: number;
  }> {
    try {
      const offset = (page - 1) * limit;
      
      // Get users with pagination
      const usersList = await db
        .select()
        .from(users)
        .orderBy(desc(users.createdAt))
        .limit(limit)
        .offset(offset);
      
      // Get total count
      const [{ count }] = await db
        .select({ count: sql<number>`count(*)` })
        .from(users);
      
      const totalPages = Math.ceil(count / limit);
      
      return {
        users: usersList,
        total: count,
        page,
        totalPages,
      };
    } catch (error) {
      console.error('❌ Error fetching users:', error);
      throw new Error('Failed to fetch users');
    }
  }

  // Update user
  static async updateUser(id: number, updates: Partial<NewUser>): Promise<User | null> {
    try {
      const [updatedUser] = await db
        .update(users)
        .set({
          ...updates,
          updatedAt: new Date(),
        })
        .where(eq(users.id, id))
        .returning();
      
      console.log('✅ User updated:', updatedUser?.username);
      return updatedUser || null;
    } catch (error) {
      console.error('❌ Error updating user:', error);
      throw new Error('Failed to update user');
    }
  }

  // Delete user (soft delete by deactivating)
  static async deactivateUser(id: number): Promise<boolean> {
    try {
      const [deactivatedUser] = await db
        .update(users)
        .set({
          isActive: false,
          updatedAt: new Date(),
        })
        .where(eq(users.id, id))
        .returning({ id: users.id });
      
      console.log('✅ User deactivated:', deactivatedUser?.id);
      return !!deactivatedUser;
    } catch (error) {
      console.error('❌ Error deactivating user:', error);
      throw new Error('Failed to deactivate user');
    }
  }

  // Hard delete user
  static async deleteUser(id: number): Promise<boolean> {
    try {
      const [deletedUser] = await db
        .delete(users)
        .where(eq(users.id, id))
        .returning({ id: users.id });
      
      console.log('✅ User deleted:', deletedUser?.id);
      return !!deletedUser;
    } catch (error) {
      console.error('❌ Error deleting user:', error);
      throw new Error('Failed to delete user');
    }
  }

  // Search users
  static async searchUsers(searchTerm: string): Promise<User[]> {
    try {
      const searchPattern = `%${searchTerm}%`;
      
      const searchResults = await db
        .select()
        .from(users)
        .where(
          or(
            ilike(users.username, searchPattern),
            ilike(users.email, searchPattern),
            ilike(users.firstName, searchPattern),
            ilike(users.lastName, searchPattern)
          )
        )
        .orderBy(desc(users.createdAt))
        .limit(20);
      
      return searchResults;
    } catch (error) {
      console.error('❌ Error searching users:', error);
      throw new Error('Failed to search users');
    }
  }

  // Get active users
  static async getActiveUsers(): Promise<User[]> {
    try {
      const activeUsers = await db
        .select()
        .from(users)
        .where(eq(users.isActive, true))
        .orderBy(desc(users.lastLoginAt));
      
      return activeUsers;
    } catch (error) {
      console.error('❌ Error fetching active users:', error);
      throw new Error('Failed to fetch active users');
    }
  }

  // Update login count and last login
  static async recordLogin(id: number): Promise<void> {
    try {
      await db
        .update(users)
        .set({
          loginCount: sql`${users.loginCount} + 1`,
          lastLoginAt: new Date(),
          updatedAt: new Date(),
        })
        .where(eq(users.id, id));
      
      console.log('✅ Login recorded for user:', id);
    } catch (error) {
      console.error('❌ Error recording login:', error);
      throw new Error('Failed to record login');
    }
  }
}
```

### Post Operations

```typescript
// src/services/postService.ts
import { eq, and, or, like, ilike, desc, asc, sql, count } from 'drizzle-orm';
import { db } from '../db/connection';
import { posts, type Post, type NewPost } from '../schema/posts';
import { users } from '../schema/users';
import { postTags, tags } from '../schema/categories';

export class PostService {
  // Create a new post
  static async createPost(postData: NewPost): Promise<Post> {
    try {
      const [newPost] = await db
        .insert(posts)
        .values({
          ...postData,
          createdAt: new Date(),
          updatedAt: new Date(),
        })
        .returning();
      
      console.log('✅ Post created:', newPost.title);
      return newPost;
    } catch (error) {
      console.error('❌ Error creating post:', error);
      throw new Error('Failed to create post');
    }
  }

  // Get post by ID with author information
  static async getPostById(id: number): Promise<(Post & { author: { username: string; firstName: string | null; lastName: string | null } }) | null> {
    try {
      const [post] = await db
        .select({
          id: posts.id,
          title: posts.title,
          slug: posts.slug,
          content: posts.content,
          excerpt: posts.excerpt,
          authorId: posts.authorId,
          viewCount: posts.viewCount,
          likeCount: posts.likeCount,
          isPublished: posts.isPublished,
          publishedAt: posts.publishedAt,
          createdAt: posts.createdAt,
          updatedAt: posts.updatedAt,
          author: {
            username: users.username,
            firstName: users.firstName,
            lastName: users.lastName,
          },
        })
        .from(posts)
        .innerJoin(users, eq(posts.authorId, users.id))
        .where(eq(posts.id, id))
        .limit(1);
      
      return post || null;
    } catch (error) {
      console.error('❌ Error fetching post by ID:', error);
      throw new Error('Failed to fetch post');
    }
  }

  // Get published posts with pagination
  static async getPublishedPosts(page: number = 1, limit: number = 10): Promise<{
    posts: (Post & { author: { username: string; firstName: string | null; lastName: string | null } })[];
    total: number;
    page: number;
    totalPages: number;
  }> {
    try {
      const offset = (page - 1) * limit;
      
      // Get posts with author information
      const postsList = await db
        .select({
          id: posts.id,
          title: posts.title,
          slug: posts.slug,
          content: posts.content,
          excerpt: posts.excerpt,
          authorId: posts.authorId,
          viewCount: posts.viewCount,
          likeCount: posts.likeCount,
          isPublished: posts.isPublished,
          publishedAt: posts.publishedAt,
          createdAt: posts.createdAt,
          updatedAt: posts.updatedAt,
          author: {
            username: users.username,
            firstName: users.firstName,
            lastName: users.lastName,
          },
        })
        .from(posts)
        .innerJoin(users, eq(posts.authorId, users.id))
        .where(eq(posts.isPublished, true))
        .orderBy(desc(posts.publishedAt))
        .limit(limit)
        .offset(offset);
      
      // Get total count
      const [{ count: totalCount }] = await db
        .select({ count: count() })
        .from(posts)
        .where(eq(posts.isPublished, true));
      
      const totalPages = Math.ceil(totalCount / limit);
      
      return {
        posts: postsList,
        total: totalCount,
        page,
        totalPages,
      };
    } catch (error) {
      console.error('❌ Error fetching published posts:', error);
      throw new Error('Failed to fetch published posts');
    }
  }

  // Search posts using full-text search
  static async searchPosts(searchTerm: string): Promise<Post[]> {
    try {
      const searchResults = await db
        .select()
        .from(posts)
        .where(
          and(
            eq(posts.isPublished, true),
            or(
              sql`to_tsvector('english', ${posts.title}) @@ plainto_tsquery('english', ${searchTerm})`,
              sql`to_tsvector('english', ${posts.content}) @@ plainto_tsquery('english', ${searchTerm})`
            )
          )
        )
        .orderBy(
          sql`ts_rank(to_tsvector('english', ${posts.title} || ' ' || ${posts.content}), plainto_tsquery('english', ${searchTerm})) DESC`
        )
        .limit(20);
      
      return searchResults;
    } catch (error) {
      console.error('❌ Error searching posts:', error);
      throw new Error('Failed to search posts');
    }
  }

  // Increment view count
  static async incrementViewCount(id: number): Promise<void> {
    try {
      await db
        .update(posts)
        .set({
          viewCount: sql`${posts.viewCount} + 1`,
          updatedAt: new Date(),
        })
        .where(eq(posts.id, id));
      
      console.log('✅ View count incremented for post:', id);
    } catch (error) {
      console.error('❌ Error incrementing view count:', error);
      throw new Error('Failed to increment view count');
    }
  }

  // Get posts by author
  static async getPostsByAuthor(authorId: number): Promise<Post[]> {
    try {
      const authorPosts = await db
        .select()
        .from(posts)
        .where(eq(posts.authorId, authorId))
        .orderBy(desc(posts.createdAt));
      
      return authorPosts;
    } catch (error) {
      console.error('❌ Error fetching posts by author:', error);
      throw new Error('Failed to fetch posts by author');
    }
  }

  // Update post
  static async updatePost(id: number, updates: Partial<NewPost>): Promise<Post | null> {
    try {
      const [updatedPost] = await db
        .update(posts)
        .set({
          ...updates,
          updatedAt: new Date(),
        })
        .where(eq(posts.id, id))
        .returning();
      
      console.log('✅ Post updated:', updatedPost?.title);
      return updatedPost || null;
    } catch (error) {
      console.error('❌ Error updating post:', error);
      throw new Error('Failed to update post');
    }
  }

  // Delete post
  static async deletePost(id: number): Promise<boolean> {
    try {
      const [deletedPost] = await db
        .delete(posts)
        .where(eq(posts.id, id))
        .returning({ id: posts.id });
      
      console.log('✅ Post deleted:', deletedPost?.id);
      return !!deletedPost;
    } catch (error) {
      console.error('❌ Error deleting post:', error);
      throw new Error('Failed to delete post');
    }
  }

  // Publish post
  static async publishPost(id: number): Promise<Post | null> {
    try {
      const [publishedPost] = await db
        .update(posts)
        .set({
          isPublished: true,
          publishedAt: new Date(),
          updatedAt: new Date(),
        })
        .where(eq(posts.id, id))
        .returning();
      
      console.log('✅ Post published:', publishedPost?.title);
      return publishedPost || null;
    } catch (error) {
      console.error('❌ Error publishing post:', error);
      throw new Error('Failed to publish post');
    }
  }
}
```

## Intermediate Operations

### Transaction Management

```typescript
// src/services/transactionService.ts
import { db } from '../db/connection';
import { users, posts } from '../schema';
import { eq } from 'drizzle-orm';

export class TransactionService {
  // Create user and initial post in a transaction
  static async createUserWithPost(userData: any, postData: any) {
    try {
      const result = await db.transaction(async (tx) => {
        // Create user first
        const [newUser] = await tx
          .insert(users)
          .values(userData)
          .returning();
        
        // Create post with the new user as author
        const [newPost] = await tx
          .insert(posts)
          .values({
            ...postData,
            authorId: newUser.id,
          })
          .returning();
        
        return { user: newUser, post: newPost };
      });
      
      console.log('✅ User and post created in transaction');
      return result;
    } catch (error) {
      console.error('❌ Transaction failed:', error);
      throw new Error('Failed to create user and post');
    }
  }

  // Transfer post ownership
  static async transferPostOwnership(postId: number, fromUserId: number, toUserId: number) {
    try {
      const result = await db.transaction(async (tx) => {
        // Verify current ownership
        const [currentPost] = await tx
          .select()
          .from(posts)
          .where(eq(posts.id, postId))
          .limit(1);
        
        if (!currentPost || currentPost.authorId !== fromUserId) {
          throw new Error('Post not found or user is not the owner');
        }
        
        // Verify target user exists
        const [targetUser] = await tx
          .select()
          .from(users)
          .where(eq(users.id, toUserId))
          .limit(1);
        
        if (!targetUser) {
          throw new Error('Target user not found');
        }
        
        // Transfer ownership
        const [updatedPost] = await tx
          .update(posts)
          .set({
            authorId: toUserId,
            updatedAt: new Date(),
          })
          .where(eq(posts.id, postId))
          .returning();
        
        return updatedPost;
      });
      
      console.log('✅ Post ownership transferred');
      return result;
    } catch (error) {
      console.error('❌ Transfer transaction failed:', error);
      throw error;
    }
  }

  // Batch operations with rollback on any failure
  static async batchCreatePosts(postsData: any[]) {
    try {
      const result = await db.transaction(async (tx) => {
        const createdPosts = [];
        
        for (const postData of postsData) {
          const [newPost] = await tx
            .insert(posts)
            .values(postData)
            .returning();
          
          createdPosts.push(newPost);
        }
        
        return createdPosts;
      });
      
      console.log(`✅ Batch created ${result.length} posts`);
      return result;
    } catch (error) {
      console.error('❌ Batch creation failed:', error);
      throw new Error('Failed to create posts in batch');
    }
  }
}
```

### Advanced Filtering and Aggregation

```typescript
// src/services/analyticsService.ts
import { db } from '../db/connection';
import { users, posts } from '../schema';
import { eq, gte, lte, and, or, sql, count, sum, avg, max, min, desc, asc } from 'drizzle-orm';

export class AnalyticsService {
  // Get user statistics
  static async getUserStatistics() {
    try {
      const stats = await db
        .select({
          totalUsers: count(),
          activeUsers: count(sql`CASE WHEN ${users.isActive} = true THEN 1 END`),
          verifiedUsers: count(sql`CASE WHEN ${users.isVerified} = true THEN 1 END`),
          avgLoginCount: avg(users.loginCount),
          maxLoginCount: max(users.loginCount),
          usersThisMonth: count(sql`CASE WHEN ${users.createdAt} >= date_trunc('month', CURRENT_DATE) THEN 1 END`),
        })
        .from(users);
      
      return stats[0];
    } catch (error) {
      console.error('❌ Error fetching user statistics:', error);
      throw new Error('Failed to fetch user statistics');
    }
  }

  // Get post statistics
  static async getPostStatistics() {
    try {
      const stats = await db
        .select({
          totalPosts: count(),
          publishedPosts: count(sql`CASE WHEN ${posts.isPublished} = true THEN 1 END`),
          draftPosts: count(sql`CASE WHEN ${posts.isPublished} = false THEN 1 END`),
          totalViews: sum(posts.viewCount),
          totalLikes: sum(posts.likeCount),
          avgViews: avg(posts.viewCount),
          avgLikes: avg(posts.likeCount),
          postsThisMonth: count(sql`CASE WHEN ${posts.createdAt} >= date_trunc('month', CURRENT_DATE) THEN 1 END`),
        })
        .from(posts);
      
      return stats[0];
    } catch (error) {
      console.error('❌ Error fetching post statistics:', error);
      throw new Error('Failed to fetch post statistics');
    }
  }

  // Get top authors by post count
  static async getTopAuthors(limit: number = 10) {
    try {
      const topAuthors = await db
        .select({
          authorId: posts.authorId,
          username: users.username,
          firstName: users.firstName,
          lastName: users.lastName,
          postCount: count(posts.id),
          totalViews: sum(posts.viewCount),
          totalLikes: sum(posts.likeCount),
          avgViews: avg(posts.viewCount),
        })
        .from(posts)
        .innerJoin(users, eq(posts.authorId, users.id))
        .where(eq(posts.isPublished, true))
        .groupBy(posts.authorId, users.username, users.firstName, users.lastName)
        .orderBy(desc(count(posts.id)))
        .limit(limit);
      
      return topAuthors;
    } catch (error) {
      console.error('❌ Error fetching top authors:', error);
      throw new Error('Failed to fetch top authors');
    }
  }

  // Get posts performance metrics
  static async getPostsPerformance(dateFrom?: Date, dateTo?: Date) {
    try {
      let query = db
        .select({
          id: posts.id,
          title: posts.title,
          authorUsername: users.username,
          viewCount: posts.viewCount,
          likeCount: posts.likeCount,
          publishedAt: posts.publishedAt,
          engagementRate: sql<number>`CASE WHEN ${posts.viewCount} > 0 THEN (${posts.likeCount}::float / ${posts.viewCount}::float) * 100 ELSE 0 END`,
        })
        .from(posts)
        .innerJoin(users, eq(posts.authorId, users.id))
        .where(eq(posts.isPublished, true));
      
      if (dateFrom && dateTo) {
        query = query.where(
          and(
            gte(posts.publishedAt, dateFrom),
            lte(posts.publishedAt, dateTo)
          )
        );
      }
      
      const performance = await query
        .orderBy(desc(posts.viewCount))
        .limit(50);
      
      return performance;
    } catch (error) {
      console.error('❌ Error fetching posts performance:', error);
      throw new Error('Failed to fetch posts performance');
    }
  }

  // Get monthly growth statistics
  static async getMonthlyGrowth() {
    try {
      const userGrowth = await db
        .select({
          month: sql<string>`to_char(${users.createdAt}, 'YYYY-MM')`,
          newUsers: count(),
        })
        .from(users)
        .where(gte(users.createdAt, sql`CURRENT_DATE - INTERVAL '12 months'`))
        .groupBy(sql`to_char(${users.createdAt}, 'YYYY-MM')`)
        .orderBy(sql`to_char(${users.createdAt}, 'YYYY-MM')`);
      
      const postGrowth = await db
        .select({
          month: sql<string>`to_char(${posts.createdAt}, 'YYYY-MM')`,
          newPosts: count(),
        })
        .from(posts)
        .where(gte(posts.createdAt, sql`CURRENT_DATE - INTERVAL '12 months'`))
        .groupBy(sql`to_char(${posts.createdAt}, 'YYYY-MM')`)
        .orderBy(sql`to_char(${posts.createdAt}, 'YYYY-MM')`);
      
      return { userGrowth, postGrowth };
    } catch (error) {
      console.error('❌ Error fetching monthly growth:', error);
      throw new Error('Failed to fetch monthly growth');
    }
  }
}
```

## Indexing and Performance

### Custom Index Management

```typescript
// src/db/indexManager.ts
import { sql } from 'drizzle-orm';
import { db } from './connection';

export class IndexManager {
  // Create custom indexes for performance optimization
  static async createPerformanceIndexes() {
    try {
      // Composite index for user search
      await db.execute(sql`
        CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_search_composite 
        ON users (is_active, is_verified, created_at DESC) 
        WHERE is_active = true;
      `);
      
      // Partial index for recent posts
      await db.execute(sql`
        CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_posts_recent 
        ON posts (published_at DESC, view_count DESC) 
        WHERE is_published = true AND published_at > CURRENT_DATE - INTERVAL '30 days';
      `);
      
      // GIN index for full-text search on posts
      await db.execute(sql`
        CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_posts_fulltext 
        ON posts USING gin(to_tsvector('english', title || ' ' || content)) 
        WHERE is_published = true;
      `);
      
      // Hash index for exact email lookups
      await db.execute(sql`
        CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email_hash 
        ON users USING hash(email);
      `);
      
      // Covering index for post list queries
      await db.execute(sql`
        CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_posts_list_covering 
        ON posts (is_published, published_at DESC) 
        INCLUDE (id, title, excerpt, author_id, view_count, like_count);
      `);
      
      console.log('✅ Performance indexes created successfully');
    } catch (error) {
      console.error('❌ Error creating performance indexes:', error);
      throw new Error('Failed to create performance indexes');
    }
  }

  // Analyze index usage
  static async analyzeIndexUsage() {
    try {
      const indexStats = await db.execute(sql`
        SELECT 
          schemaname,
          tablename,
          indexname,
          idx_scan as scans,
          idx_tup_read as tuples_read,
          idx_tup_fetch as tuples_fetched,
          pg_size_pretty(pg_relation_size(indexrelid)) as size
        FROM pg_stat_user_indexes 
        ORDER BY idx_scan DESC;
      `);
      
      return indexStats.rows;
    } catch (error) {
      console.error('❌ Error analyzing index usage:', error);
      throw new Error('Failed to analyze index usage');
    }
  }

  // Find unused indexes
  static async findUnusedIndexes() {
    try {
      const unusedIndexes = await db.execute(sql`
        SELECT 
          schemaname,
          tablename,
          indexname,
          pg_size_pretty(pg_relation_size(indexrelid)) as size
        FROM pg_stat_user_indexes 
        WHERE idx_scan = 0 
        AND indexname NOT LIKE '%_pkey'
        ORDER BY pg_relation_size(indexrelid) DESC;
      `);
      
      return unusedIndexes.rows;
    } catch (error) {
      console.error('❌ Error finding unused indexes:', error);
      throw new Error('Failed to find unused indexes');
    }
  }

  // Get table statistics
  static async getTableStatistics() {
    try {
      const tableStats = await db.execute(sql`
        SELECT 
          schemaname,
          tablename,
          n_live_tup as live_tuples,
          n_dead_tup as dead_tuples,
          n_tup_ins as inserts,
          n_tup_upd as updates,
          n_tup_del as deletes,
          last_vacuum,
          last_autovacuum,
          last_analyze,
          last_autoanalyze,
          pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size
        FROM pg_stat_user_tables 
        ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
      `);
      
      return tableStats.rows;
    } catch (error) {
      console.error('❌ Error getting table statistics:', error);
      throw new Error('Failed to get table statistics');
    }
  }

  // Optimize table maintenance
  static async optimizeTableMaintenance() {
    try {
      // Analyze tables for better query planning
      await db.execute(sql`ANALYZE;`);
      
      // Get tables that need vacuum
      const vacuumNeeded = await db.execute(sql`
        SELECT tablename 
        FROM pg_stat_user_tables 
        WHERE n_dead_tup > 1000 
        OR (n_dead_tup > 0 AND n_dead_tup::float / NULLIF(n_live_tup + n_dead_tup, 0) > 0.1);
      `);
      
      // Vacuum tables that need it
      for (const row of vacuumNeeded.rows) {
        await db.execute(sql.raw(`VACUUM ANALYZE ${row.tablename};`));
        console.log(`✅ Vacuumed table: ${row.tablename}`);
      }
      
      console.log('✅ Table maintenance optimization completed');
    } catch (error) {
      console.error('❌ Error optimizing table maintenance:', error);
      throw new Error('Failed to optimize table maintenance');
    }
  }
}
```

### Query Performance Monitoring

```typescript
// src/utils/queryMonitor.ts
import { sql } from 'drizzle-orm';
import { db } from '../db/connection';

export class QueryMonitor {
  // Enable query monitoring
  static async enableQueryMonitoring() {
    try {
      // Enable pg_stat_statements if not already enabled
      await db.execute(sql`CREATE EXTENSION IF NOT EXISTS pg_stat_statements;`);
      
      // Reset statistics
      await db.execute(sql`SELECT pg_stat_statements_reset();`);
      
      console.log('✅ Query monitoring enabled');
    } catch (error) {
      console.error('❌ Error enabling query monitoring:', error);
      throw new Error('Failed to enable query monitoring');
    }
  }

  // Get slow queries
  static async getSlowQueries(limit: number = 10) {
    try {
      const slowQueries = await db.execute(sql`
        SELECT 
          query,
          calls,
          total_exec_time,
          mean_exec_time,
          stddev_exec_time,
          rows,
          100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
        FROM pg_stat_statements 
        WHERE calls > 5
        ORDER BY mean_exec_time DESC 
        LIMIT ${limit};
      `);
      
      return slowQueries.rows;
    } catch (error) {
      console.error('❌ Error getting slow queries:', error);
      throw new Error('Failed to get slow queries');
    }
  }

  // Get queries with high I/O
  static async getHighIOQueries(limit: number = 10) {
    try {
      const highIOQueries = await db.execute(sql`
        SELECT 
          query,
          calls,
          shared_blks_read + shared_blks_written as total_io,
          shared_blks_read,
          shared_blks_written,
          shared_blks_dirtied,
          100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
        FROM pg_stat_statements 
        WHERE shared_blks_read + shared_blks_written > 0
        ORDER BY total_io DESC 
        LIMIT ${limit};
      `);
      
      return highIOQueries.rows;
    } catch (error) {
      console.error('❌ Error getting high I/O queries:', error);
      throw new Error('Failed to get high I/O queries');
    }
  }

  // Monitor active queries
  static async getActiveQueries() {
    try {
      const activeQueries = await db.execute(sql`
        SELECT 
          pid,
          usename,
          application_name,
          client_addr,
          state,
          query_start,
          state_change,
          EXTRACT(EPOCH FROM (NOW() - query_start)) as duration_seconds,
          LEFT(query, 100) as query_preview
        FROM pg_stat_activity 
        WHERE state = 'active' 
        AND pid <> pg_backend_pid()
        ORDER BY query_start;
      `);
      
      return activeQueries.rows;
    } catch (error) {
      console.error('❌ Error getting active queries:', error);
      throw new Error('Failed to get active queries');
    }
  }

  // Get blocking queries
  static async getBlockingQueries() {
    try {
      const blockingQueries = await db.execute(sql`
        SELECT 
          blocked_locks.pid AS blocked_pid,
          blocked_activity.usename AS blocked_user,
          blocking_locks.pid AS blocking_pid,
          blocking_activity.usename AS blocking_user,
          blocked_activity.query AS blocked_query,
          blocking_activity.query AS blocking_query
        FROM pg_catalog.pg_locks blocked_locks
        JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
        JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
          AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
          AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
          AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
          AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
          AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
          AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
          AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
          AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
          AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
          AND blocking_locks.pid != blocked_locks.pid
        JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
        WHERE NOT blocked_locks.granted;
      `);
      
      return blockingQueries.rows;
    } catch (error) {
      console.error('❌ Error getting blocking queries:', error);
      throw new Error('Failed to get blocking queries');
    }
  }
}
```

## Advanced Queries

### Complex Joins and Subqueries

```typescript
// src/services/advancedQueryService.ts
import { db } from '../db/connection';
import { users, posts, categories, tags, postTags } from '../schema';
import { eq, and, or, sql, exists, notExists, inArray, desc, asc } from 'drizzle-orm';

export class AdvancedQueryService {
  // Get posts with all related data using complex joins
  static async getPostsWithAllData(limit: number = 10) {
    try {
      const postsWithData = await db
        .select({
          // Post data
          postId: posts.id,
          title: posts.title,
          slug: posts.slug,
          excerpt: posts.excerpt,
          viewCount: posts.viewCount,
          likeCount: posts.likeCount,
          publishedAt: posts.publishedAt,
          
          // Author data
          author: {
            id: users.id,
            username: users.username,
            firstName: users.firstName,
            lastName: users.lastName,
          },
          
          // Aggregated tag data
          tagCount: sql<number>`COUNT(DISTINCT ${postTags.tagId})`,
          tags: sql<string>`STRING_AGG(DISTINCT ${tags.name}, ', ' ORDER BY ${tags.name})`,
        })
        .from(posts)
        .innerJoin(users, eq(posts.authorId, users.id))
        .leftJoin(postTags, eq(posts.id, postTags.postId))
        .leftJoin(tags, eq(postTags.tagId, tags.id))
        .where(eq(posts.isPublished, true))
        .groupBy(posts.id, users.id)
        .orderBy(desc(posts.publishedAt))
        .limit(limit);
      
      return postsWithData;
    } catch (error) {
      console.error('❌ Error fetching posts with all data:', error);
      throw new Error('Failed to fetch posts with all data');
    }
  }

  // Get users with their post statistics using window functions
  static async getUsersWithPostStats() {
    try {
      const usersWithStats = await db
        .select({
          userId: users.id,
          username: users.username,
          firstName: users.firstName,
          lastName: users.lastName,
          totalPosts: sql<number>`COUNT(${posts.id})`,
          publishedPosts: sql<number>`COUNT(CASE WHEN ${posts.isPublished} = true THEN 1 END)`,
          totalViews: sql<number>`COALESCE(SUM(${posts.viewCount}), 0)`,
          totalLikes: sql<number>`COALESCE(SUM(${posts.likeCount}), 0)`,
          avgViews: sql<number>`COALESCE(AVG(${posts.viewCount}), 0)`,
          maxViews: sql<number>`COALESCE(MAX(${posts.viewCount}), 0)`,
          lastPostDate: sql<Date>`MAX(${posts.publishedAt})`,
          authorRank: sql<number>`RANK() OVER (ORDER BY COUNT(${posts.id}) DESC)`,
          viewsRank: sql<number>`RANK() OVER (ORDER BY COALESCE(SUM(${posts.viewCount}), 0) DESC)`,
        })
        .from(users)
        .leftJoin(posts, eq(users.id, posts.authorId))
        .groupBy(users.id)
        .orderBy(desc(sql`COUNT(${posts.id})`));
      
      return usersWithStats;
    } catch (error) {
      console.error('❌ Error fetching users with post stats:', error);
      throw new Error('Failed to fetch users with post stats');
    }
  }

  // Get posts with similar content using subqueries
  static async getSimilarPosts(postId: number, limit: number = 5) {
    try {
      // First get the target post's tags
      const targetPostTags = db
        .select({ tagId: postTags.tagId })
        .from(postTags)
        .where(eq(postTags.postId, postId));
      
      const similarPosts = await db
        .select({
          id: posts.id,
          title: posts.title,
          slug: posts.slug,
          excerpt: posts.excerpt,
          viewCount: posts.viewCount,
          publishedAt: posts.publishedAt,
          authorUsername: users.username,
          commonTags: sql<number>`COUNT(DISTINCT ${postTags.tagId})`,
          similarity: sql<number>`
            COUNT(DISTINCT ${postTags.tagId}) * 100.0 / 
            NULLIF((SELECT COUNT(*) FROM post_tags WHERE post_id = ${postId}), 0)
          `,
        })
        .from(posts)
        .innerJoin(users, eq(posts.authorId, users.id))
        .innerJoin(postTags, eq(posts.id, postTags.postId))
        .where(
          and(
            eq(posts.isPublished, true),
            sql`${posts.id} != ${postId}`,
            inArray(postTags.tagId, targetPostTags)
          )
        )
        .groupBy(posts.id, users.username)
        .having(sql`COUNT(DISTINCT ${postTags.tagId}) > 0`)
        .orderBy(desc(sql`COUNT(DISTINCT ${postTags.tagId})`))
        .limit(limit);
      
      return similarPosts;
    } catch (error) {
      console.error('❌ Error fetching similar posts:', error);
      throw new Error('Failed to fetch similar posts');
    }
  }

  // Get trending posts using complex calculations
  static async getTrendingPosts(days: number = 7, limit: number = 10) {
    try {
      const trendingPosts = await db
        .select({
          id: posts.id,
          title: posts.title,
          slug: posts.slug,
          viewCount: posts.viewCount,
          likeCount: posts.likeCount,
          publishedAt: posts.publishedAt,
          authorUsername: users.username,
          
          // Trending score calculation
          trendingScore: sql<number>`
            (
              (${posts.viewCount} * 0.6) + 
              (${posts.likeCount} * 2.0) + 
              (CASE 
                WHEN ${posts.publishedAt} > CURRENT_DATE - INTERVAL '1 day' THEN 50
                WHEN ${posts.publishedAt} > CURRENT_DATE - INTERVAL '3 days' THEN 25
                WHEN ${posts.publishedAt} > CURRENT_DATE - INTERVAL '7 days' THEN 10
                ELSE 0
              END)
            ) / 
            GREATEST(EXTRACT(EPOCH FROM (CURRENT_TIMESTAMP - ${posts.publishedAt})) / 3600, 1)
          `,
          
          // Engagement rate
          engagementRate: sql<number>`
            CASE 
              WHEN ${posts.viewCount} > 0 
              THEN (${posts.likeCount}::float / ${posts.viewCount}::float) * 100 
              ELSE 0 
            END
          `,
        })
        .from(posts)
        .innerJoin(users, eq(posts.authorId, users.id))
        .where(
          and(
            eq(posts.isPublished, true),
            sql`${posts.publishedAt} > CURRENT_DATE - INTERVAL '${days} days'`
          )
        )
        .orderBy(
          desc(sql`
            (
              (${posts.viewCount} * 0.6) + 
              (${posts.likeCount} * 2.0) + 
              (CASE 
                WHEN ${posts.publishedAt} > CURRENT_DATE - INTERVAL '1 day' THEN 50
                WHEN ${posts.publishedAt} > CURRENT_DATE - INTERVAL '3 days' THEN 25
                WHEN ${posts.publishedAt} > CURRENT_DATE - INTERVAL '7 days' THEN 10
                ELSE 0
              END)
            ) / 
            GREATEST(EXTRACT(EPOCH FROM (CURRENT_TIMESTAMP - ${posts.publishedAt})) / 3600, 1)
          `)
        )
        .limit(limit);
      
      return trendingPosts;
    } catch (error) {
      console.error('❌ Error fetching trending posts:', error);
      throw new Error('Failed to fetch trending posts');
    }
  }

  // Get content recommendations using collaborative filtering
  static async getContentRecommendations(userId: number, limit: number = 10) {
    try {
      // This is a simplified collaborative filtering approach
      // In production, you might want to use more sophisticated algorithms
      
      const recommendations = await db
        .select({
          id: posts.id,
          title: posts.title,
          slug: posts.slug,
          excerpt: posts.excerpt,
          viewCount: posts.viewCount,
          likeCount: posts.likeCount,
          publishedAt: posts.publishedAt,
          authorUsername: users.username,
          recommendationScore: sql<number>`
            (
              ${posts.viewCount} * 0.3 + 
              ${posts.likeCount} * 0.7
            ) * 
            (
              1 + (
                SELECT COUNT(*) * 0.1 
                FROM post_tags pt1 
                WHERE pt1.post_id = ${posts.id} 
                AND pt1.tag_id IN (
                  SELECT DISTINCT pt2.tag_id 
                  FROM posts p2 
                  JOIN post_tags pt2 ON p2.id = pt2.post_id 
                  WHERE p2.author_id = ${userId}
                )
              )
            )
          `,
        })
        .from(posts)
        .innerJoin(users, eq(posts.authorId, users.id))
        .where(
          and(
            eq(posts.isPublished, true),
            sql`${posts.authorId} != ${userId}`
          )
        )
        .orderBy(desc(sql`
          (
            ${posts.viewCount} * 0.3 + 
            ${posts.likeCount} * 0.7
          ) * 
          (
            1 + (
              SELECT COUNT(*) * 0.1 
              FROM post_tags pt1 
              WHERE pt1.post_id = ${posts.id} 
              AND pt1.tag_id IN (
                SELECT DISTINCT pt2.tag_id 
                FROM posts p2 
                JOIN post_tags pt2 ON p2.id = pt2.post_id 
                WHERE p2.author_id = ${userId}
              )
            )
          )
        `))
        .limit(limit);
      
      return recommendations;
    } catch (error) {
      console.error('❌ Error fetching content recommendations:', error);
      throw new Error('Failed to fetch content recommendations');
    }
  }

  // Get posts with hierarchical categories
  static async getPostsWithHierarchicalCategories() {
    try {
      const postsWithCategories = await db.execute(sql`
        WITH RECURSIVE category_hierarchy AS (
          -- Base case: categories without parents
          SELECT 
            id, 
            name, 
            slug, 
            parent_id, 
            name as full_path,
            0 as level
          FROM categories 
          WHERE parent_id IS NULL
          
          UNION ALL
          
          -- Recursive case: categories with parents
          SELECT 
            c.id, 
            c.name, 
            c.slug, 
            c.parent_id,
            ch.full_path || ' > ' || c.name as full_path,
            ch.level + 1 as level
          FROM categories c
          JOIN category_hierarchy ch ON c.parent_id = ch.id
        )
        SELECT 
          p.id,
          p.title,
          p.slug,
          p.published_at,
          u.username as author,
          ch.full_path as category_path,
          ch.level as category_level
        FROM posts p
        JOIN users u ON p.author_id = u.id
        LEFT JOIN category_hierarchy ch ON p.category_id = ch.id
        WHERE p.is_published = true
        ORDER BY p.published_at DESC;
      `);
      
      return postsWithCategories.rows;
    } catch (error) {
      console.error('❌ Error fetching posts with hierarchical categories:', error);
      throw new Error('Failed to fetch posts with hierarchical categories');
    }
  }
}