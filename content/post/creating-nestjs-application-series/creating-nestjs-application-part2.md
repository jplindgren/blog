---
date: 2020-05-20T18:00:08-03:00
description: "Creating a simple but complete backend blog application with NestJS, Postgres and ElasticSearch"
featured_image: "/images/nestjs-logo-small.svg"
series: ["Complete NestJS"]
tags: [ "Javascript", "Nestjs", "Elasticsearch", "TypeORM"]
title: "Creating a NestJS application (Part II)"
weight: 70
slug: part2
---
[Part I]({{< relref "creating-nestjs-application-part1.md" >}})

Part II
- [Add Database Persistence](#add-persistence-with-typeorm)
- [Configuration](#add-configuration-service)
- Docker and Docker compose
- Add a post subscriber and index content with elastic search.

In part I of this series in our NestJs application we created a post entity, a post services, a couple of endpoints to manage it and finally the authentication using google OAuth. While it was a lot of work, we are far from having a sample of usable application. We are still missing a core part of most of the web applications, the data persistence. In this part we will make the persistence of our data using a library called [TypeORM](https://typeorm.io/) and the [postgres](https://www.postgresql.org/) database. Because some sensitive configuration parameters like username and password are needed by the application, the next section will be dedicated to the configuration of the application. How can we remove these hardcoded parameters from our application and how we should deal with them?
As a result, at this point we should have a small functional sample application. We have entities, validation, endpoints, authentication, persistence and configuration. In addition to this, in the following steps I´ll show how to [dockerize](https://docs.docker.com/get-started/) our application and add elastic search to index the posts, making it easier to search and provide suggestion about related content.
In the future, I´ll do a part III of this series implementing a frontend in [react](https://reactjs.org/) to use our backend.

## Add Persistence with TypeORM
For this application the postgres will be the choosen database. Feel free to change and choose another one of you preference. We are going to use a library called [TypeORM](https://typeorm.io/) and accordingly with their documentation they support a lot of databases, like mysql, mariadb, mssql, mongodb and others. The TypeORM as the name implies is an [Object-relational mapping](https://en.wikipedia.org/wiki/Object-relational_mapping) tool and you probably knows what it is. But in case you are completely lost, in short, it is a technique that maps the relational world of our database tables, to the object world, in our application represented by the entities and its associations. We are not needed to write any sql (although we can for custom complex scenarios or optmizations), to save, list, delete and so on our entities, the only work is to configure how will be our mappings and the library will do the dirty part of the job under the hoods, generating the queries, managing transactions, etc.
TypeORM can work with [Repository](https://martinfowler.com/eaaCatalog/repository.html) pattern or [ActiveRecord](https://martinfowler.com/eaaCatalog/activeRecord.html) pattern. Again, if you have no clue what are these patterns, I suggest you to do a quick look on the links or if you can do better, read the book [Patterns of Enterprise Application Architecture](https://www.martinfowler.com/books/eaa.html).
For this demo, I am choosing the ActiveRecord way, firstly because I think is more suited for small applications, but also for a matter of preference.
I will make an assumption that you already have you database installed and configured, but if you don't just follow the guide and wait until the docker section where we will use an postgres image to create a container with our database. No database installation on your machine will be needed.

First lets install typeorm dependencies and the postgres drive.

    PS D:\projects\bloggering> npm i --save @nestjs/typeorm typeorm postgres

Next, import and configure the `TypeOrmModule`in the `AppModule` (the root module of the system).

**bloggering/src/app.module.ts**

	import { Module } from  '@nestjs/common';
	import { AppController } from  './app.controller';
	import { AppService } from  './app.service';
	import { PostsModule } from  './posts/posts.module';
	import { AuthModule } from  './auth/auth.module';
	import { TypeOrmModule } from  '@nestjs/typeorm';

	@Module({
		imports: [
			PostsModule, AuthModule, TypeOrmModule.forRoot({
				type:  'postgres',
				host:  'localhost',
				port:  5432,
				username:  '{username}',
				password:  '{password}',
				database:  'bloggering',
				entities: ['dist/**/*.entity.js'],
				synchronize:  true,
			})
		],
		controllers: [AppController],
		providers: [AppService],
	})
	export  class  AppModule { }
Later we will remove these sensitive parameters from the code and inject it into the application. Also it should be noted that we imported the TypeOrmModule in the root module of the application, hence we can inject the typeorm objects through the entire without import any other module.
In the entities option we used a glob pattern to get all our entities on the project. So every time we add one new entity, it ill not be necessary to do any changes to this option.
The last thing to notice in this configuration, but one of the most important is about the `syncronize: true`. Enabling this option make TypeORM auto update your schema everytime it changes, IT IS NOT RECOMMENDED to have this option in production for example because you can lose your data. In a real project you will probably work with migrations, I´ll not include it here because this series is getting bigger enough. Be aware that if you want to use TypeORM CLI to generate your migrations you should put your configurations into your `.env` or `ormconfig.json` file. But keep calm, in the next section we will move all of our config settings to an `.env` file and configure a `ConfigService`.
If you want to learn more about migrations there are some links below.
Oficial documentation
[https://typeorm.io/#/migrations](https://typeorm.io/#/migrations)
Contains best practices to initialize the migrations in a new app.
[https://github.com/typeorm/typeorm/issues/2961](https://github.com/typeorm/typeorm/issues/2961)
Some weird behavior to TypeORM CLI get configurations from env file.
[https://github.com/typeorm/typeorm/issues/4288](https://github.com/typeorm/typeorm/issues/4288)

Next add the mappings decorators to the entities and its properties. These decorators will be responsible to tell TypeORM how to map our objects to the database schema.

**bloggering/src/users/user.entity.ts**

	import { BaseEntity, Column, Entity, Unique, CreateDateColumn, UpdateDateColumn, VersionColumn, PrimaryGeneratedColumn } from  'typeorm';

	@Entity()
	@Unique(['email'])
	@Unique(['thirdPartyId'])
	export  class  User  extends  BaseEntity {
		@PrimaryGeneratedColumn('uuid')
		id: string;

		@Column()
		thirdPartyId: string;

		@Column({ length:  100 })
		name: string;

		@Column({ length:  120 })
		email: string;

		@Column()
		isActive: boolean;

		@CreateDateColumn({ name:  'created_at' })
		createdAt!: Date;

		@UpdateDateColumn({ name:  'updated_at' })
		UpdatedAt!: Date;

		@VersionColumn()
		version!: number;
	}

**bloggering/src/posts/post.entity.ts**

	import { IsNotEmpty } from  'class-validator';
	import {
	Entity,
	Column,
	PrimaryGeneratedColumn,
	BaseEntity,
	ManyToOne,
	} from  'typeorm';
	import { User } from  'src/users/user.entity';

	@Entity()
	export  class  Post  extends  BaseEntity {
		constructor(author: User, title: string, content: string) {
			super();
			this.author = author;
			this.title = title;
			this.content = content;
		}

		@PrimaryGeneratedColumn('uuid')
		id: string;

		@ManyToOne(type  =>  User, { eager:  true })
		author: User;

		@Column('varchar', { length:  255 })
		@IsNotEmpty()
		title: string;

		@Column('text')
		@IsNotEmpty()
		content: string;
	}
The changes were for the most part is just simple stuff, we decorated the two entities with the `@Entity()` decorator to tell TypeORM that this is and entity. If we want, we can even customize the names of the table that will be generated passing a string as parameter to this decorator. Subsequently we decorated the entitiy's properties with the self explanatory @Column decorator and each specific column configuration.
In the `Post` entity the author property was changed from a string to be a real association with the `User` object. The `@ManyToOne` states that can be Many posts for a single user, and as an additional parameter we set `eager` to true, which means that everytime we query for a post, the user will be loaded along.
Aditionally we extended our entities from the `BaseEntity` class. This is only needed if you want to follow the **ActiveRecord** pattern, otherwise you can create a repository to manage your entity, take a look [here](https://typeorm.io/#/working-with-repository) if this is your preference . In this case, because of the ActiveRecord we will be able to call static methods in our class to manipulate data in the database.
To retrieve the active users for instance we can use:

	const activeUsers =  await User.find({ isActive:  true  });

You can take a look in all the methods os the `BaseEntity` class [here](https://typeorm.delightful.studio/classes/_repository_baseentity_.baseentity.html)

We are almost there! Now, just update the `post.service.ts` and the `post.controller.ts` to call our active records method to save, retrieve real data from the database and reflect our changes in model.
After the changes the files should looks like:

**bloggering/src/posts/posts.service.ts**

	import { Injectable, ForbiddenException } from  '@nestjs/common';
	import { Post } from  './post.entity';
	import { User } from  'src/users/user.entity';

	@Injectable()
	export  class  PostsService {
		async  createPost(title: string, author: User, content: string): Promise<any> {
			const  insertedPost = await  Post.save(new  Post(author, title, content));
			return  insertedPost.id
		}
		getPosts = async (): Promise<Post[]> =>  await  Post.find();
		getSinglePost = async (id: string): Promise<Post> =>  await  Post.findOneOrFail(id);
	}

**bloggering/src/posts/posts.controller.ts**

```typescript {linenos=inline,hl_lines=[8,"15-17"],linenostart=199}
import { Post  as  BlogPost } from  './post.entity';
import { PostsService } from  './posts.service';
import { Controller, Get, Param, Post, Body, UseGuards, Req } from  '@nestjs/common';
import { AuthGuard } from  '@nestjs/passport/dist';

@Controller('posts')
export  class  PostsController {
    constructor(private  readonly  postsService: PostsService) { }

    @Get()
    async  findAll(): Promise<BlogPost[]> {
        return  await  this.postsService.getPosts();
    }

    @Get(':id')
    async  find(@Param('id') postId: string): Promise<BlogPost> {
        return  await  this.postsService.getSinglePost(postId);
    }

    @Post()
    @UseGuards(AuthGuard('jwt'))
    async  create(@Body() newPost: BlogPost, @Req() req) {
    const  user = req['user'];
    const  newPostId = await  this.postsService.createPost(newPost.title, user, newPost.content);
    return { id:  newPostId };
    }
}
```

In the `PostsService` we changed the methods to be `async` return promises and call the active records methods from our `Post` entity (because we extended `BaseEntity` remember?
Moreover, we had to do little changes the `PostsController` as well. The methods were turned into `async` and the returns `Promise<>`. The `create`method passes the whole user instead of just the user name.
Now, create your database (we used in this tutorial the name blogguering), you can do it by hand or use a script. Later, when use docker it will be created automatically for us.
We are ready to test our app. If for some reason you need to debug and is using VSCode, make sure to install `ts-node` locally  (even if you have it globally installed) and add the follow `launch.json` inside the `.vscode` folder on the root of your application. Usually this file is not versioned and each developer can have its own debug setup.

**bloggering/src/.vscode/launch.json**

{{< gist jplindgren bccfa2209733eba4af09cda30f511eb8 >}}

Let's try to create a new post to test what we´ve done.
Oops... An error.
![NestJs scaffold structure](/images/create-nestjs-app-part2/01-error-on-save-post.png)
This error happened because we are setting the post with an author that is not persisted. Remember that we set in the post mapping a relation between the post and user (author). So the TypeORM expect a created User as the right association. We are going to fix it putting a logic to create a user if it does not exists in the `AuthService`. In a more sofitsticate application you will probably want to segregate this logic into a `UserService` but for us here, let's keep it simple.
Change the `validateOAuthLogin` in the `AuthService` to be like this:
**bloggering/src/auth/auth.service.ts**

	...
	async  validateOAuthLogin(email: string, name: string, thirdPartyId: string, provider: string): Promise<string> {
		try {
			let  user: User = await  User.findByThirdPartyId(thirdPartyId);
			if (!user) {
				user = new  User();
				user.isActive = true;
				user.email = email;
				user.name = name;
				user.thirdPartyId = thirdPartyId;
				user = await  User.save(user);
			}
			return  this.jwtService.sign({
				id:  user.id,
				provider
			});
		}
		catch (err) {
			throw  new  InternalServerErrorException('validateOAuthLogin', err.message);
		}
	}
	...

Along with the creation of the `findByThirdPartyId` method in the `User` entity.
**bloggering/src/users/user.entity.ts**

	...
	static  findByThirdPartyId(thirdPartyId: string): Promise<User> {
		return  this.createQueryBuilder("user")
			.where("user.thirdPartyId = :thirdPartyId", { thirdPartyId })
			.getOne();
	}
	...
Basically, now there is a check when the user is being logged. If the user does not exists we create one, and due to using Google OAuth as login we check it by the google profile Id, thus we added a new method to retrieve a user by thirdPartyId into our `User` entity.

The last piece we are goingo to change to make it work, is our JwtStrategy. This time, the `validate` method will do a query in the database to get the real user, check permissions and return.

**bloggering/src/auth/strategies/jwt.strategy.ts**

	...
	async  validate(payload: any) {
		const  user = await  User.findOneOrFail(payload.id);
		//validate token, user claims, etc
		return  user;
	}
	...

Finally! Now we can authenticate with our google login, create a post and retrieve it. We can restart the application as many times as we want and the data will still be there! Go check the database to look at our new post.
![NestJs scaffold structure](/images/create-nestjs-app-part2/02-post-in-database.png)

## Add Configuration Service