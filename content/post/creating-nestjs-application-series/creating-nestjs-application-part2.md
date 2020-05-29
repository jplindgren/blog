---
date: 2020-05-20T18:00:08-03:00
description: "Creating a simple but complete backend blog application with NestJS, Postgres and ElasticSearch"
featured_image: "/images/nestjs-logo-small.svg"
series: ["Complete NestJS"]
tags: [ "Javascript", "Nestjs", "Elasticsearch", "TypeORM", "Docker"]
title: "Creating a NestJS application (Part II)"
summary: "Part II of the series 'Creating a NestJs application'. In this part we are going to add TypeORM to save our entities to the database, deal with the application settings in a more safe way, run our application and all its dependencies in containers using docker and docker-compose, and in the end, we are going to index our content to elastic search providing blazing fasts searches!"
weight: 90
slug: part2
---
BE AWARE THAT THIS IS A WORK IN PROGRESS... I UPLOADED AS PUBLISHED TO HELP PEOPLE AND GET FEEDBACKS.
IF YOU WANT TO HELP ME WITH FEEDBACKS, YOU CAN SEND AN EMAIL TO joaopozo@gmail.com

[Part I]({{< relref "creating-nestjs-application-part1.md" >}})

Part II
- [Add Database Persistence](#add-persistence-with-typeorm)
- [Configuration](#add-configuration-service)
- [Docker and Docker compose](#add-docker-and-docker-compose)
- [Add a post subscriber and index content with elastic search](#add-elastic-search)

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
```typescript {linenos=inline}
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
```
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
```typescript {linenos=inline}
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
```

**bloggering/src/posts/post.entity.ts**
```typescript {linenos=inline}
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
```
The changes were for the most part is just simple stuff, we decorated the two entities with the `@Entity()` decorator to tell TypeORM that this is and entity. If we want, we can even customize the names of the table that will be generated passing a string as parameter to this decorator. Subsequently we decorated the entitiy's properties with the self explanatory @Column decorator and each specific column configuration.
In the `Post` entity the author property was changed from a string to be a real association with the `User` object. The `@ManyToOne` states that can be Many posts for a single user, and as an additional parameter we set `eager` to true, which means that everytime we query for a post, the user will be loaded along.
Aditionally we extended our entities from the `BaseEntity` class. This is only needed if you want to follow the **ActiveRecord** pattern, otherwise you can create a repository to manage your entity, take a look [here](https://typeorm.io/#/working-with-repository) if this is your preference . In this case, because of the ActiveRecord we will be able to call static methods in our class to manipulate data in the database.
To retrieve the active users for instance we can use:

	const activeUsers =  await User.find({ isActive:  true  });

You can take a look in all the methods os the `BaseEntity` class [here](https://typeorm.delightful.studio/classes/_repository_baseentity_.baseentity.html)

We are almost there! Now, just update the `post.service.ts` and the `post.controller.ts` to call our active records method to save, retrieve real data from the database and reflect our changes in model.
After the changes the files should looks like:

**bloggering/src/posts/posts.service.ts**
```typescript {linenos=inline}
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
```
**bloggering/src/posts/posts.controller.ts**

```typescript {linenos=inline}
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

```typescript {linenos=inline, linenostart=10, hl_lines=[3, 5, 10]}
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
```

Along with the creation of the `findByThirdPartyId` method in the `User` entity.
**bloggering/src/users/user.entity.ts**

```typescript {linenos=inline, linenostart=31}
static  findByThirdPartyId(thirdPartyId: string): Promise<User> {
	return  this.createQueryBuilder("user")
		.where("user.thirdPartyId = :thirdPartyId", { thirdPartyId })
		.getOne();
}
```
Basically, now there is a check when the user is being logged. If the user does not exists we create one, and due to using Google OAuth as login we check it by the google profile Id, thus we added a new method to retrieve a user by thirdPartyId into our `User` entity.

The last piece we are goingo to change to make it work, is our JwtStrategy. This time, the `validate` method will do a query in the database to get the real user, check permissions and return.

**bloggering/src/auth/strategies/jwt.strategy.ts**

```typescript {linenos=inline, linenostart=16}
async  validate(payload: any) {
	const  user = await  User.findOneOrFail(payload.id);
	//validate token, user claims, etc
	return  user;
}
```

Finally! Now we can authenticate with our google login, create a post and retrieve it. We can restart the application as many times as we want and the data will still be there! Go check the database to look at our new post.
![NestJs scaffold structure](/images/create-nestjs-app-part2/02-post-in-database.png)

## Add Configuration Service

After speaking the whole text about take care with sensitive information hardcoded in our code it is time to get rid of them. Not only sensitive information to be honest, we'll also change configurations that can be possible changed by multiple enviroments, like hosts, secret keys, etc.
Most of the readers already know about lots of strategies to deal with it, most of the languages encourages use enviroments variables, others uses config files, but the porpose is always the same. Keep your config data out of you code making possible to change settings according to the environment . NodeJS usually uses .env files for store settings, and NestJS follow the same approach but adds a infrastructure over the top o process.env to help with modularity and testability of the code. They call it [ConfigService](https://docs.nestjs.com/techniques/configuration#using-the-configservice) and we are going to use it to create our custom configuration files.

 - Application Configuration
 - Authentication Configuration
 - Database configuration

Clearly this is an overengineer for a small project like this. Usually you need to split things in this granularity in big projects, but as I wanted to explore the options I choose to break it. Feel free to follow the documentation example and load directly the env file into the ConfigService and inject is as Global if you think is more appropriated for your project. It is up to you define what is better in YOUR application.
First and foremost we need to install NestJs config dependency.

	PS C:\gems\blog-projects\bloggering> npm i  @nestjs/config --save

Then let's create an `/config` folder under the `/src` folder. Then inside the /config folder create one folder for each of the groups we talked above, `/app`, `/auth`, `/database`
![NestJs scaffold structure](/images/create-nestjs-app-part2/03-config-folders-structure.png)

Now in the root of your project create a .env file to hold our configs. This file should not be versioned! Check if it is already ignored in the .gitignore file for safety.
Inside the enviroment file we can add all of our configs.

	APP_ENV=development
	APP_NAME=Bloggering
	APP_PORT=3000
	APP_HOST=http://localhost
	GOOGLE_OAUTH2_CLIENTID=<your_client_id>
	GOOGLE_OAUTH2_CLIENT_SECRET=<your_client_secret>
	OAUTH2_JWT_SECRET=<your_jwt_secret>
	TYPEORM_CONNECTION=postgres
	TYPEORM_HOST=localhost
	TYPEORM_USERNAME=postgres
	TYPEORM_PASSWORD=<your_postgres_password>
	TYPEORM_DATABASE=bloggering
	TYPEORM_PORT=5432
	TYPEORM_LOGGING=true
	TYPEORM_ENTITIES=dist/**/*.entity.js
	TYPEORM_SYNCHRONIZE=true
Of course, feel free to change them according with your enviroment. For example, my postgres is exposed in the port 5432, but your can be in another port.
To deal with the general application configuration we are going to create a `AppConfigService`and expose it via an `AppConfigModule`
Create three files inside `/src/config/app` folder: `configuration.ts`,  `configuration.service.ts`, and `configuration.module.ts`.

**bloggering/src/config/app/configuration.ts**

 ```typescript {linenos=inline}
import { registerAs } from  '@nestjs/config';
export  default  registerAs('app', () => ({
	env:  process.env.APP_ENV,
	name:  process.env.APP_NAME,
	host:  process.env.APP_HOST,
	port:  process.env.APP_PORT,
}));
```
The `app/configuration.ts` is solely responsible to load the configurations of the env file and create a namespace for them. To do that, we call the factory `registerAs` from `@nestjs/config`.

**bloggering/src/config/app/configuration.service.ts**

 ```typescript {linenos=inline}
import { Injectable } from  '@nestjs/common';
import { ConfigService } from  '@nestjs/config';

@Injectable()
export  class  AppConfigService {
	constructor(private  configService: ConfigService) { }
	get  name(): string {
		return  this.configService.get<string>('app.name');
	}
	get  env(): string {
		return  this.configService.get<string>('app.env');
	}
	get  host(): string {
		return  this.configService.get<string>('app.host');
	}
	get  port(): number {
		return  Number(this.configService.get<number>('app.port'));
	}
}
```

The `AppConfigService` is our actual custom settings service. Our class is basically a wrapper on top of the ConfigService (injected via constructor) and it will be exposed via a module allowing us to do a partial registration of this settings to only specific features if we want.

**bloggering/src/config/app/configuration.module.ts**
 ```typescript {linenos=inline}
import { Module } from  '@nestjs/common';
import  configuration  from  './configuration';
import { AppConfigService } from  './configuration.service';
import { ConfigModule, ConfigService } from  '@nestjs/config';

@Module({
	imports: [
		ConfigModule.forRoot({
			load: [configuration],
		}),
	],
	providers: [ConfigService, AppConfigService],
	exports: [ConfigService, AppConfigService],
})
export  class  AppConfigModule { }
```
The last piece of infrastructure to the app's configuration settings is the module.
Through it we'll be able to allow our features to import our custom `AppConfigService`.

To exemplify how to use, let's remove the hardcoded port in the `main.ts` file for our port from the env file. But to be able to use it there, make sure to first import the `AppConfigModule` in our `AppModule`.

**bloggering/src/app.module.ts**
 ```typescript {linenos=inline, hl_lines=[4]}
@Module({
imports: [
PostsModule, AuthModule,
AppConfigModule, ...],
...
})
 ```
 Finally we can change our main.ts to set the port based on our settings. Due to be the entrypoint of the NestJS we cannot inject the `AppConfigService` on the bootstrap function, but we can use the the application to retrieve our custom configuration. Check the code below.

**bloggering/src/main.ts**
```typescript {linenos=inline, hl_lines=[4, 10, 11]}
import { NestFactory } from  '@nestjs/core';
import { AppModule } from  './app.module';
import { ValidationPipe } from  '@nestjs/common';
import { AppConfigService } from  './config/app/configuration.service';

async  function  bootstrap() {
	const  app = await  NestFactory.create(AppModule);
	app.useGlobalPipes(new  ValidationPipe());
	app.setGlobalPrefix('api');
	const  appConfig: AppConfigService = app.get('AppConfigService');
	await  app.listen(appConfig.port || 3000);
}
bootstrap();
```
If everything worked as we wish, you should be able to change `APP_PORT` settings in our `.env` file and the application should start to listen in this new port.

The database configuration service needs some customization because the `OrmConfigService` needs to implements the `TypeOrmOptionsFactory` interface. We need to implement the `createTypeOrmOptions` that returns a `TypeOrmModuleOptions` to fullfill the contract that TypeORM expects. Looking at the code make it easier to understand.

**bloggering/src/config/database/configuration.ts**
```typescript {linenos=inline, hl_lines=[2]}
import { registerAs } from  '@nestjs/config';
const  entitiesPath = process.env.TYPEORM_ENTITIES ? `${process.env.TYPEORM_ENTITIES}` : 'dist/**/*.entity.js';

export  default  registerAs('orm', () => ({
	type:  process.env.TYPEORM_CONNECTION,
	host:  process.env.TYPEORM_HOST || '127.0.0.1',
	username:  process.env.TYPEORM_USERNAME,
	password:  process.env.TYPEORM_PASSWORD,
	database:  process.env.TYPEORM_DATABASE,
	logging:  process.env.TYPEORM_LOGGING === 'true',
	sincronize:  process.env.TYPEORM_SYNCHRONIZE === 'true',
	port:  Number.parseInt(process.env.TYPEORM_PORT, 10),
	entities: [entitiesPath],
}));
```
We added some default values if no settings is provided and some especial treatments for arrays, booleans and numbers. We gave these settings the namespace of "orm".

**bloggering/src/config/database/configuration.service.ts**
```typescript {linenos=inline, hl_lines=[6, 8]}
import { Injectable } from  '@nestjs/common';
import { ConfigService } from  '@nestjs/config';
import { TypeOrmModuleOptions, TypeOrmOptionsFactory } from  '@nestjs/typeorm';

@Injectable()
export  class  OrmConfigService  implements  TypeOrmOptionsFactory {
	constructor(private  configService: ConfigService) { }
	createTypeOrmOptions(): TypeOrmModuleOptions {
		const  type: any = this.configService.get<string>('orm.type');
		return {
			type,
			host:  this.configService.get<string>('orm.host'),
			username:  this.configService.get<string>('orm.username'),
			password:  this.configService.get<string>('orm.password'),
			database:  this.configService.get<string>('orm.database'),
			port:  this.configService.get<number>('orm.port'),
			logging:  this.configService.get<boolean>('orm.logging'),
			entities:  this.configService.get<string[]>('orm.entities'),
			synchronize:  this.configService.get<boolean>('orm.sincronize'),
		};
	}
}
```
Besides that,  there is one point that is worth to talk. How to change our hardcoded settings for the new settings in the TypeORM configuration module on `app.module.ts`.
Currently we call the method `forRoot` of the `TypeOrmModule`:
 ```typescript {linenos=false}
 TypeOrmModule.forRoot({
	type:  'postgres',
	host:  'localhost',
	port:  5432,
	username:  'postgres',
	password:  '<password>',
	database:  'bloggering',
	entities: [Post, User],
	synchronize:  true,
})
 ```

The problem is that method receives static configuration, to remediate, we need to change to call the method `forRootAsync` which we can provide a  `TypeOrmOptionsFactory` that returns a `TypeOrmModuleOptions`. That why our `OrmConfigService` needs to implement this exact interface. Now the TypeORM will know how to get the settings from our custom database config service.
Check the `AppModule` after the changes.

**bloggering/src/app_module.ts**
```typescript {linenos=inline, hl_lines=["12-15"]}
import { Module } from  '@nestjs/common';
import { AppService } from  './app.service';
import { PostsModule } from  './posts/posts.module';
import { AuthModule } from  './auth/auth.module';
import { TypeOrmModule } from  '@nestjs/typeorm';
import { AppConfigModule } from  './config/app/configuration.module';
import { OrmConfigModule } from  './config/database/configuration.module';
import { OrmConfigService } from  './config/database/configuration.service';

@Module({
	imports: [PostsModule, AuthModule, AppConfigModule, OrmConfigModule,
		TypeOrmModule.forRootAsync({
			imports: [OrmConfigModule],
			useClass:  OrmConfigService
		})],
	controllers: [],
	providers: [AppService],
})
export  class  AppModule { }
  ```

From now on, I´ll speed up the velocity because for the rest of the settings, the procedures are basically the same, we are going to create a configuration, a custom config service and a module for outh settings and any other "feature" our logic block we want. You can check their content in the [bloggering project](https://github.com/jplindgren/Bloggering) repository on GitHub. Do not forget to change the places where we used this hardcoded settings to inject the corresponding config service.
Of course, as I said in the beggining of the section, this is a worth strategy only for big projects, for small projects you can just call to load you env file automatically. You can even set it global and inject all over the places.
```typescript
@Module({
  imports: [ConfigModule.forRoot()],
})
```

Finally, just run the application and check if everything is still working fine. We are ready for the next part and docker and docker compose to our stack!

## Add Docker and Docker Compose

In this section we are going to configure our application ro run via [Docker](https://docs.docker.com/), and also configure the docker compose to automatically get the images of the postgres and later of the elastic search and then run everything togheter without install anything manually.

If you already know what docker is, or its purpose you can skip the next paragraph and go to the real action, otherwise, I tried to write a briefly instrocution about the it.

Until now, we are starting our application on our local machine and acessing the database installed on localhost. But what if you want to execute it in other machine? For instance, your notebook? Or other member of you team wants to run the application as well? You/He is going to execute all the steps manually, install node, install dependencies, install and run the database in your machine. You can create scripting for everything, even so, is tedious and susceptible to bugs, the script can be wrong, the envinroment can have differences, OS, libs, database version, etc. Not only is a possible problem for development, but also when you want to deploy your application. You develop everything in your machine, works fine. Then, you have ship your code to production and boom! A completely diferent environment there, and you of course, wonder what the heck happened? In my machine it worked like a charm!
In conclusion, these are the kinds of problems we can solve with containers. And there comes Docker to rescue. Using it, we can pack, ship, and run any application as a lightweight, portable, self-sufficient container, which can run virtually anywhere. Creating CI/CD pipelines will be much more easier because with this self container you have predictable, and isolation of you application.

First and foremost, install docker for your OS. Just a quick search in google and you will find the link. This install docker for windows, go to [https://docs.docker.com/docker-for-windows/install/](https://docs.docker.com/docker-for-windows/install/).

First create a [dockerfile](https://docs.docker.com/engine/reference/builder/) in the root of the application.
 ```dockerfile {linenos=inline}
FROM  node:12.13-alpine As development
ENV  NODE_ENV=development
WORKDIR  /usr/src/app
COPY  package*.json  ./
RUN  npm  install  --only=development
COPY  .  .
RUN  npm  run  build

FROM  node:12.13-alpine as production
ARG  NODE_ENV=production
ENV  NODE_ENV=${NODE_ENV}
WORKDIR  /usr/src/app
COPY  package*.json  ./
RUN  npm  install  --only=production
COPY  .  .
COPY  --from=development  /usr/src/app/dist  ./dist
CMD  ["node",  "dist/main"];
 ```
 This is a [multi stage](https://docs.docker.com/develop/develop-images/multistage-build/) build file, with a stage we called development and other called production.  In a multi-stage build like that, we will have a much lightweight image.
 The development step is used to generate our javascript files in the /dist folder. If you have no clue about what I'm talking about I suggest you to google about typescript. But long story short, typescript is a superset of javascript, but in the end, needs to be compiled to plain javascript. What NestJs Build does under the hoods, is to call typescript compiler to compile the files to javascript. You can check [here](https://docs.nestjs.com/cli/scripts#build).
 - Development step
	 - The `FROM` command is the beginning of a stage. We specified an image
   to be used and we give a name to that step. This image will be used
   until our next `FROM` command which will result in another step with
   another image.
	  - The `WORKDIR` command sets the working directory for the subsequents commands COPY, RUN, and CMD. The first `COPY` copies our `package.json` and `package.lock.json` to our workdir, then we run `npm install` with the flag to install only dev dependencies. The next `COPY` copies the rest of the application files to our workdir in our container. Finally we call the `RUN` command to run `npm run build` and build our application generating our `/dist` folder.
  - Production step
	  - The next `FROM` creates another stage using a fresh new image with a different name. Both of the stages have no connections between them.
	  - The nexts instructions are very similar with the previous stage, with some differences. When we run `npm install` we use the `--only=prod` flag to not install dev dependencies.
	  - We have one more `COPY` command to copy from the `development` stage the `/dist` content. And finally the `CMD` set the default command to execute when the image is run.

 With the dockerfile created we are ready to build our image running the command above.

	 PS C:\gems\blog-projects\bloggering> docker build -t bloggering-api .
 The `-t` means a tag, we give the name bloggering-api.
![NestJs scaffold structure](/images/create-nestjs-app-part2/04-building-docker-image.png)

 You can check if the image was created with the command `docker images` in your terminal.
 Now we can run our container. Make sure to use the same tag you used to build the image.
 ```script
	docker run bloggering-api
```

But wait, there is a problem, we are running our application in a container, if you have a local database you will not have access by default. To overcome this, you have some solutions that you can see [here](https://superuser.com/questions/1254515/setup-a-docker-container-to-work-with-a-local-database), but well... we are not going to use any of them. Instead, we will use a tool called [Docker Compose](https://docs.docker.com/compose/) that comes with docker.
Docker Compose allow us to define and run more than one container, set their configurations and dependencies between them. Imagine that you have a big application with multiple microservices, a database, a message queue, and any other service you could imagine. Instead of start each one of them manually, we just configure our docker-compose to run one container for each of our services and which depends of which one and voilá! With a single magical command we start all of them.
Let's see it in practice.
Similarly as we did before with dockerfile, now create a `docker-compose.yml` file in the root of our application.
![NestJs scaffold structure](/images/create-nestjs-app-part2/05-docker-compose-file-organization.png)

**docker-compose.yml**
 ```yml
version: '3.7'
services:
	postgres:
		container_name: ${TYPEORM_HOST}
		image: postgres:12-alpine
		networks:
		- blognetwork
		env_file:
		- .env
		environment:
			POSTGRES_PASSWORD: ${TYPEORM_PASSWORD}
			POSTGRES_USER: ${TYPEORM_USERNAME}
			POSTGRES_DB: ${TYPEORM_DATABASE}
			PG_DATA: /var/lib/postgresql/data
		ports:
			- ${TYPEORM_PORT}:${TYPEORM_PORT}
		volumes:
			- pgdata:/var/lib/postgresql/data

	main:
		container_name: bloggering-api
		restart: unless-stopped
		build:
			context: .
			target: development
		command: npm run start:dev
		ports:
			- ${APP_PORT}:${APP_PORT}
		volumes:
			- .:/usr/src/app
			- /usr/src/app/node_modules
		env_file:
			- .env
		networks:
			- blognetwork
		depends_on:
			- postgres

networks:
	blognetwork:
volumes:
	pgdata:
 ```
A quick look on this file should give you a clue about what is happening here. We created two services (will have two separated containers in our application), one for the postgres database and the other for our API.
The first service is our database, postgres. We specified the container name to be the same as our TYPEORM_HOST. We do this because instead of acessing localhost now, we are acessing this database in a container from another container in the same network. And by default, the database host is the container_name.  Then we set the image, the same named network we will set to the **main service** and the file we have our enviroment variables. In envinroment section we set pre defined postgres's variables. You can check the complete list [here](https://hub.docker.com/_/postgres).
The two last configurations is about the ports (**Host_Port:Container_Port**) we want to expose and the named volume we will set. Volumes are the way we can persist our data even when recreate a image for example.
The main service has the container name set as bloggering-api. The build section we defined the  `context` which tells Docker which files should be sent to the Docker daemon. In our case, that’s our whole application, and so we pass in `.`, which means all of the current directory. The `target` will match our stage in the dockerfile and made docker ignore the production stage.
The `command` set what command we will run in the container start, because it is a development enviroment, we will run it with the watch to allow reload the application everytime something changes. The `volumes` part has some tricks, because we want to docker to reflect any change we make in our files we mount our host current directory into the Docker container. Due to this volume we can have our changes reflected in our application. With this in mind, we have to take care to our node_modules do not override the container node_modules. In order to avoid that, we mount an anonymous volume to avoid that behavior.
All the rest is self explanatory, we set the enviroment file, the network (same as the postgres service), and we tell docker compose that this service depends on the database service. So Docker Compose will try to start first our postgres service, and then, our bloggering_api service.

> If you already configure your databases settings, do not forget to
> change TYPEORM_HOST to other value than localhost.

Now we can run the command to run docker-compose.

	PS C:\gems\blog-projects\bloggering> docker-compose up

Docker compose will start both our servers, in the correct order.

**When you need to install new dependencies via `npm` or `yarn` you´ll have to run the follow command.**

	PS C:\gems\blog-projects\bloggering> docker-compose up --build -V
Remember when we created our `docker-compose.yml` file, we set an anonymous volume to our node_modules in our API to avoid it to be overriden. The -build flag triggers a `npm install` and the `-V` will remove this anonymous volume and recreated it.

However you may notice that because our application now runs via docker-compose and we call `npm run start:dev`  we cannot debug it anymore. In order to fix that problem, let's change some pieces.
>I´ll put here the steps to debug in **VSCODE** the IDE that I use.

Change the docker-compose.yml file to `run npm run start:debug` instead also lets add the default VSCode debbuger port when using the `--inspect` flag, `9229`.

**docker-compose.yml**
 ```yml {lines_hl=[5, 8]}
	 services:
	 ...
		 main:
			 ...
			command: npm run start:debug
			ports:
				- ${APP_PORT}:${APP_PORT}
				- 9229:9229
			...
```

In your `package.json` file change the debug inside the scripts

**package.json**

	"start:debug": "nest start --debug 0.0.0.0:9229 --watch",

Last but no least replace our `.vscode/launch.json` with the below content.
 ```json {lines_hl=[5, 8]}
{
	"version": "0.2.0",
	"configurations": [
		{
			"type": "node",
			"request": "attach",
			"name": "Debug: Bloggering",
			"remoteRoot": "/usr/src/app",
			"localRoot": "${workspaceFolder}",
			"protocol": "inspector",
			"port": 9229,
			"restart": true,
			"address": "0.0.0.0",
			"skipFiles": [
				"<node_internals>/**"
			]
		}
	]
}
 ```

After that changes, put a breakpoint in you code, `run docker-compose up` command and try to attach the debugger to see if it works.
Hopefully everything worked fine and now we have our envinroment all set working in containers! Yeah!
Let's move on to the next and final step of our backend project in NestJS!

## Add Elastic Search
In shortly elasticsearch is a REST HTTP service that wraps around [Apache Lucene](https://lucene.apache.org/) adding scalability and making it distributed. Lucene is powerfull Java-based indexing and search technology. Its use cases varies from indexing millions of log entries to indexing ecommerce content, blogs, streaming tweets, show related content, and many others.
For this example, we are going to use it to index our posts (title, content, author) and provide endpoints to do quick search and get related content.

Install the [NestJs Elasticsearch](https://github.com/nestjs/elasticsearch) module and the elasticsearch.

	PS C:\gems\blog-projects\bloggering> npm i @elastic/elasticsearch --save
	PS C:\gems\blog-projects\bloggering> npm i @nestjs/elasticsearch --save

Because we already have our configuration pattern defined using custom configuration files, let's do the same for the elasticsearch, if you skipped this custom configuration feel free to skip this part and do whatever you did before. Create a folder called `elasticsearch` inside our `config` folder, inside the folder create our 3 basic files, `configuration.ts`, `configuration.service.ts`, `configuration.module.ts`. I won´t enter in detail because we already looked into this part on the [Configuration](#add-configuration-service) section.
The structure should looks like:
![NestJs scaffold structure](/images/create-nestjs-app-part2/06-elasticsearch-config-structure.png)

**bloggering/src/config/configuration.module.ts**
 ```typescript
import { registerAs } from  '@nestjs/config';
export  default  registerAs('es', () => ({
	host:  process.env.ELASTIC_SEARCH_HOST || '127.0.0.1',
	port:  process.env.ELASTIC_SEARCH_PORT || '9200',
}));
 ```

**bloggering/src/config/configuration.service.ts**
```typescript {linenos=inline}
import { Injectable } from  '@nestjs/common';
import { ConfigService } from  '@nestjs/config';
import { ElasticsearchOptionsFactory, ElasticsearchModuleOptions } from  '@nestjs/elasticsearch';

@Injectable()
export  class  ElasticsearchConfigService  implements  ElasticsearchOptionsFactory {
	constructor(private  configService: ConfigService) { }

	createElasticsearchOptions(): ElasticsearchModuleOptions {
		const  node = `http://${this.configService.get<string>('es.host')}:${this.configService.get<string>('es.port')}`;
		return { node };
	}
}
```
>Likewise the TypeORM configuration, the elasticsearch module also accepts a specific interface that has a method that returns `ElasticsearchModuleOptions`.

**bloggering/src/config/configuration.module.ts**
```typescript {linenos=inline}
import { Module } from  '@nestjs/common';
import  configuration  from  './configuration';
import { ElasticsearchConfigService } from  './configuration.service';
import { ConfigModule, ConfigService } from  '@nestjs/config';

@Module({
	imports: [
		ConfigModule.forRoot({
			load: [configuration]
		}),
	],
	providers: [ConfigService, ElasticsearchConfigService],
	exports: [ConfigService, ElasticsearchConfigService],
})
export  class  ElasticsearchConfigModule { }
```
### Creating a subscriber
A subscriber is an object that can be attached to the TypeORM pipeline and listen to certains events in an entity. This object should implement an interface called EntitySubscriberInterface. Common events including beforeInsert, afterInsert, beforeUpdate, afterUpdate an others can be implemented.
Subscribers sometimes overlaps with another feature from TypeORM called `@Listeneres`, where you can decorate custom methods in the entity itself to listen for specific events as well.
Although, it is a cool feature, seems to have some limitations, like we cannot inject dependencies into this methods, and we cannot conditionally subscribe or not like it is possible when using subscribers.

| Subscribers|
|--|--|
| (+) Can be subscribed/or listen dynamically |
| (+) Can inject dependencies |
| (+) Can listen to all entities |
| (-) More complex, need to be subscribed and registered as provider |

| Listeners|
|--|--|
| (+) Easy to implement |
| (+) Can use the own property of the entity, increasing cohesion |
| (-) Can't be applied dynamically |
| (-) Can't have injected dependencies |

The documentation instructs to add subscribers to the TypeORM configuration as a [Glob](https://en.wikipedia.org/wiki/Glob_%28programming%29) pattern, but we are not going to do that. Letting TypeORM take care of it  makes it impossible for us to use dependency injection into our subscriber, that's why instead, we´ll subscribe it manually on the constructor to make NestJS take care of the injection. It is a bit hacky to me, but is the only way I found to inject dependencies right now in the subscriber. You can check discussions [here](https://github.com/nestjs/typeorm/pull/27#issuecomment-431296683) and [here](https://github.com/nestjs/typeorm/issues/112).

TODO: Add DOCKER_COMPOSE CONFIG and .env CONFIG

Create the `post.subrscriber.ts` file inside the `/posts` folder.
**bloggering/src/posts/post.subscriber.ts**
```typescript {linenos=inline, hl_lines=[8]}
import { EventSubscriber, EntitySubscriberInterface, Connection, InsertEvent, UpdateEvent } from  "typeorm";
import { Post } from  "./post.entity";
import { ElasticsearchService } from  '@nestjs/elasticsearch';

@EventSubscriber()
export  class  PostSubscriber  implements  EntitySubscriberInterface<Post> {
	constructor(private  readonly  connection: Connection, private  readonly  elasticSearchService: ElasticsearchService) {
		connection.subscribers.push(this);
	}

	listenTo() {
		return  Post;
	}

	async  afterInsert(event: InsertEvent<Post>) {
		await  this.createOrUpdateIndexForPost(event.entity);
	}

	afterUpdate(event: UpdateEvent<Post>) {
		console.log(`After Post Updated: `, event);
	}

	async  createOrUpdateIndexForPost(post: Post) {
		const  indexName = Post.indexName();
		try {
			await  this.elasticSearchService.index({
				index:  indexName,
				id:  post.id,
				body:  post.toIndex(),
			});
		} catch (error) {
			console.log(error);
		}
	}
}
```
It is important implement the method `ListenTo`, otherwise the subscriber will listen to all entities configured in the TypeORM configuration. The method `afterInsert`  will be fired after a `Post` insert/save and will receive the entity just inserted. Then, we call the method `createOrUpdateIndexForPost`which is the responsible to call elasticsearch and index our post information. The name of the index will be get from a post entity helper method as well as the body of the document. (documents can be would be like the rows in relational databases).
   > Notice that our index post will be created after the first call if it does not exists.

Under the hoods, all the elasticseach client do for us, is call REST endpoints of the elasticsearch server. You can call them manually for tests if you want.
For example,  after build the elasticsearch image try to execute a curl command in the docker container to check if elasticsearch is running.

    docker exec -it es01 curl http://localhost:9200

So far so good,  now add the helpers methods to the `Post` entity. In the end of the file add the methods `indexName()` and `toIndex()`.
**bloggering/src/posts/post.entity.ts**
```typescript {linenos=inline}
static  indexName() {
	return  Post.name.toLowerCase();
}

toIndex() {
	return {
	id:  this.id,
	author:  this.author.name,
	title:  this.title,
	content:  this.content
}
```

In our `PostModule` add the elasticsearch configuration and the PostSubscriber as a provider.
**bloggering/src/posts/post.entity.ts**
```typescript {linenos=inline, hl_lines=[11,"12-15" ]}
import { Module } from  '@nestjs/common';
import { PostsController } from  './posts.controller';
import { PostsService } from  './posts.service';
import { ElasticsearchModule } from  '@nestjs/elasticsearch';
import { ElasticsearchConfigService } from  'src/config/elasticsearch/configuration.service';
import { ElasticsearchConfigModule } from  'src/config/elasticsearch/configuration.module';
import { PostSubscriber } from  './posts.subscriber';

@Module({
	controllers: [PostsController],
	providers: [PostsService, PostSubscriber],
	imports: [ElasticsearchModule.registerAsync({
		imports: [ElasticsearchConfigModule],
		useClass:  ElasticsearchConfigService
	})]
})
export  class  PostsModule { }
```

From now on, the index part should be working. Try to run the docker-compose to check if it works. In case some problem happen, try to look at NestJs log and debug the application if needed. Double check the docker-compose.yml configuration and enviroment file. Make sure you are not using localhost or the wrong port.
Call elasticsearch mannualy to check if it is working.

	    docker exec -it es01 curl http://localhost:9200

In case everything worked fine, you can execute the commands above to check the result:

	    docker exec -it es01 curl http://localhost:9200/post/
	    docker exec -it es01 curl http://localhost:9200/post/_search
The first one should return to us the schema of our document post, and the second the result of a search.

By the way, the search will be the last piece of code we are going to implement. The Bloggering application will have one endpoint to ultra fast search by keywords and other to get related content.

Create a method search in the `PostsService` class.
**bloggering/src/posts/post.service.ts**
```typescript {linenos=inline, hl_lines=[8, 10, 12, "17-18" ]}
import { Injectable, ForbiddenException } from  '@nestjs/common';
import { Post } from  './post.entity';
import { User } from  'src/users/user.entity';
import { ElasticsearchService } from  '@nestjs/elasticsearch';

@Injectable()
export  class  PostsService {
	constructor(private  readonly  elasticSearchService: ElasticsearchService) { }

	async  search(options: any): Promise<Post> {
		const { body } = await  this.elasticSearchService.search({
			index:  Post.indexName(),
			body: {
				query: {
				// eslint-disable-next-line @typescript-eslint/camelcase
					multi_match: {
						query:  options.query,
						fields: ['title^2', 'content', 'author'],
					}
				}
			}
		});
		return  body.hits;
	}

	async  createPost(title: string, author: User, content: string): Promise<any> {
		if (!author.isActive)
			throw  new  ForbiddenException("Cannot create a post because the user is inactive");
		const  insertedPost = await  Post.save(new  Post(author, title, content));
		return  insertedPost.id
	}
	getPosts = async (): Promise<Post[]> =>  await  Post.find();
	getSinglePost = async (id: string): Promise<Post> =>  await  Post.findOneOrFail(id);
}
```
Our new `search` method receives an argument `options` that contains a property called `query` with the **keyword** searched by the client. Then, calls `elasticSearchService.search` passing as arguments, the name of the index, and an object containing the expression searched by the user (**options.query**) and the an array of fields where we´ll make the search. Notice that we can use the expression `^` to increase the rate of a specific field. In my case, title has its weighte increase so will have a bigger impact in the query.
Diferent from others databases, elasticsearch have the possibility of instead have a binary return (found, not found), return a score for the results, the higher the score, the more likely to be a relevant result.

Time to expose our `search` via an endpoint in our application. After that, our clients will be able to query for posts in a quick way!
Just add a search method with a specific route in the `post.controller.ts`
**bloggering/src/posts/post.controller.ts**
```typescript {linenos=inline}
import { Post  as  BlogPost } from  './post.entity';
import { PostsService } from  './posts.service';
import { Controller, Get, Param, Post, Body, UseGuards, Req, Query } from  '@nestjs/common';
import { AuthGuard } from  '@nestjs/passport/dist';

@Controller('posts')
export  class  PostsController {
	constructor(private  readonly  postsService: PostsService) { }

	@Get()
	async  findAll(): Promise<BlogPost[]> {
		return  await  this.postsService.getPosts();
	}

	@Get('search')
	async  search(@Query('q') content): Promise<BlogPost> {
		return  this.postsService.search({ query:  content });
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


TODO: More detailed query
As regards the related content... TODO






## References:

[https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html)

[https://www.elastic.co/guide/en/elasticsearch/reference/current/docs.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs.html)
[https://www.elastic.co/guide/en/elasticsearch/client/javascript-api/current/api-reference.html](https://www.elastic.co/guide/en/elasticsearch/client/javascript-api/current/api-reference.html)
