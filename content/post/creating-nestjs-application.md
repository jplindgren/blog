---
date: 2020-05-14T10:58:08-04:00
description: "Creating a sample blog application with NestJS, Postgres and ElasticSearch"
featured_image: "/images/nestjs-logo-small.svg"
tags: [ "Javascript", "Nestjs", "Elasticsearch"]
title: "Creating a NestJS application"
---
# Creating a "complete" NestJS application

- [What is NestJS](#what-is-nestjs)
- [Creating a Project](#creating-a-project)
- [Adding controllers](#add-controllers)
- Adding services
- Adding Google Auth
- Http test
- Add TypeORM
- Configuration
- Add a post subscriber and index content with elastic search.
- Docker and Docker compose

During this pandemic with the free time, I and some friends wanted to do a side project to improve our skills and know new technologies and frameworks. I´ll not focus on what would be this project, but after a few talks we decided to do it, using react native and NodeJs in the backend.
It turns out that we did not take this side project out of paper for several reasons, conciliate schedules, different objectives, and lack of agreement as to the direction in which the project should go, among other things. During our discussions one of my friends presented this framework called NestJS as an option to use in the backend. Although we halt the project, I decided to give it a shot to this framework, and started a little web application on my own just to learn it.

This article describes in detail the steps I took in creating a "fake" blog API. I start by explaining what NestJS is, how you can set it up, add controllers, persist data, and dockerize it. Be warned that this is just a SAMPLE PROJECT, and while some parts are kept very simple or even lacking, in other aspects you will see some OVERENGINEER because I wanted to discover how things work in NestJS.

## What is NestJS
As you can see [here](https://docs.nestjs.com/), the documentation claims NestJS to be a framework for building efficient, scalable NodeJs applications. It uses progressive JavaScript and combines OOP (Object Oriented Programming), FP (Functional Programming), and FRP (Functional Reactive Programming). Putting the ~~fancy words~~ buzzwords aside, it creates an abstraction on top of [express](https://expressjs.com/) (or [fastify](https://www.fastify.io/)) providing an opinionated application **achitecture**. I kind of like opinionated frameworks, although you can customize things, and do it your way, they kind of strongly guide you to use certain tools and code using their style as you can see looking into their documentation. Because of that, create something from scratch is super fast and you can jump in a new project without spending much time thinking how you are going to organize things, "it is not so easy" to break the rules and create messy code using it. A nice thing to have especially for newcomers to the framework. Also they use typescript which is very positive to me (I confess that I had preconception, but after using it for some time, I fell in love with it) and seems to have a decent CLI.
One thing that can be a positive point for some, is that it looks a lot like [Angular.js](https://angularjs.org/). As Angular is not my background, it was indifferent to me, but I think that it should be very straightforward for angular developers to jump into NestJS.

## Creating the project

Well, I decided to create a ~~complex and inovative~~ old but gold sample project! An API for a blog, where we will be able to retrieve, list, create, update, and publish posts! Exciting no!? =]
Jokes apart, I think we can use a lot of tools creating a sample project like this, and to be honest, it was the first idea that came to my mind to create an article.
I am using Windows, Windows Terminal, and Node v14.2.0.

** Let's install the CLI (Command-line interface) package and scaffold a new project.**

    PS D:\projects> npm i -g @nestjs/cli
    PS D:\projects> nest new bloggering
    PS D:\projects\bloggering> cd bloggering

We just installed the CLI globally and use it to scaffold our new project with the default arguments (you can use args to change language, package manager, etc). The CLI already create a folder for us with the basic project structure. The entry point of the application is the ***main.ts*** where we create our application and define the port it will listen.
If you want this is the place where you can set a global prefix for your project. It could be for example.

    const app = await NestFactory.create(AppModule);
    app.setGlobalPrefix('api');

Opening the folder in an IDE its structure should looks like.
![NestJs scaffold structure](/images/create-nestjs-app/01-project-structure-after-post-module.png)

We can run the application now to check it.

    PS D:\projects\bloggering> npm run start

You can use fiddler, postman, curl or even the browser and navigate to http://localhost:3000 to see a "hello world* message.

## Add controllers
Lets get into the action and add our fist controller. We we´ll create a posts controller that will be responsible to handle all the basic operations over a Post, like create, edit, list, get etc.
First lets create a posts module where we´ll use to organize everything related with posts into it. Modules are the way NestJS uses to organize your application dependencies keeping each feature segregated and following SOLID principles. Every NestJS has at least one module, the root module. Lets use the CLI to create one for us using the command generate with the argument ***module*** (mo).
Also lets create a controller as well.

    PS D:\projects\bloggering> nest generate mo posts
    PS D:\projects\bloggering> nest generate co posts

![NestJs scaffold structure](/images/create-nestjs-app/02-project-structure-after-create.png)
Look that nest created a folder called **posts** with a *posts.controller.ts* and *posts.module.ts* for us. Besides that, it also updated our *app.module.ts* to import our new **PostsModule**.
In the src/posts folder create a file ***post.entity.ts*** that will contain our post entity.
After that Inside our new posts.controller.ts we´ll create a couple of endpoints to deal with requests to /posts routes. In the end we should have these two files.

**bloggering/src/posts/post.entity.ts**

		export  class  Post {
			constructor(author: string, title: string, content: string) {
				this.author = author;
				this.title = title;
				this.content = content;
			}
			id: number;
			author: string;
			title: string;
			content: string;
		}

**bloggering/src/posts/posts.controller.ts**

	import { Controller, Get, Param, Post, HttpCode, Body } from  '@nestjs/common';
	import { Post  as  BlogPost } from  './post.entity';

	@Controller('posts')
	export  class  PostsController {

		@Get()
		findAll(): BlogPost[] {
			const  posts = [new  BlogPost("me", "a post", "post content")];
			return  posts;
		}

		@Get(':id')
		find(@Param('id') postId: number): BlogPost {
			const  post = new  BlogPost("me", "a post", "post content");
			post.id = postId;
			return  post;
		}

		@Post()
		@HttpCode(204)
		create(@Body() newPost: BlogPost) {
		}
	}
Until now everything is pretty straightforward. Just basic endpoints with some fake data. Later we are going to create DTOs to avoid expose inner objects to the external world and save and retrieve real posts. By the time the only points worth of notice are:

- I had to import the Post entity as PostEntity. I had to do that,
because the name collides with the Post method decorator from NestJS.
- In the *find* method we defined a route param prefixing with ':' like **:id**. We can access its value decorating the argument received by the method with "@Param('**id**'). Of course the param defined in the route and the decorator has to match.
- In the *create* method we set the HTTP return status as 204 (no content). And although we did not use it yet, we decorated our argument newPost with the @Body. This means that the content of the body of the HTTP request will be mapped to this field.

Run your application and try to access our news endpoints at post controller. http://localhost:3000/api/posts. If you don´t know you can use start:dev to run in watch mode. Using this we can run our application and reloads our application automatically every time we modify and save a file.

	PS D:\projects\bloggering> npm run start:dev

NestJs show us every route mapped in the build output on terminal.
![NestJs scaffold structure](/images/create-nestjs-app/03-terminal-with-routes.png)
Both Post and Get verbs are already responding
![NestJs scaffold structure](/images/create-nestjs-app/04-post-api-calls.png)
In the next section we are going to create a post service and start to deal with real posts!

## Add services
Currently we have our controllers just creating hardcoded data to return from its endpoints. Now we are going to create a PostsService to save/retrieve dynamically created post entities. But, they will not be saved in the database yet, instead we´ll save them in memory for a while.
After create this service, we will need to include it as a provider in the *PostsModule*, this will tell NestJS injector the scope of this dependency and help it control its instantiation and encapsulation to other parts of the system. Then we can inject the service into the PostsController and use it to retrieve and create our posts.
To automatically update the PostsModule dependencies, lets use the CLI to generate our service. As before, the command pattern are the same, we just have to change the schematics name to service.

	PS D:\projects\bloggering> nest generate service posts
NestJS will automatically create a ***posts.service.ts*** file inside the posts folder for us, and update the provider of our PostsModule.
Later we are add [TypeORM](https://typeorm.io/#/) to  persist our posts in a database, but for now, the application will handle the posts in memory. Lets code our PostsService class.

**bloggering/src/posts/posts.service.ts**

		import { Injectable } from  '@nestjs/common';
		import { Post } from  './post.entity';

	@Injectable()
	export  class  PostsService {
		private  posts: any = {};
		private  currentId = 1;

		createPost(title: string, authorId: string, content: string){
			const  post = new  Post(authorId, title, content);
			post.id = this.currentId;
			this.posts[post.id] = post;
			this.currentId++;
			return { id:  post.id };
		}

		getPosts = (): Array<Post> =>  Object.values(this.posts);
		getSinglePost = (id: number): Post  =>  this.posts[id];
	}
Pretty simple right? We imported the post entity (by the way, because of the typescript awesomeness, in some IDEs like [VSCode](https://code.visualstudio.com/) if its not imported you can simply press ctrl+. and choose the option to import Post entity from module *post.entity*), then we decorate our class with the @Injectable() marking it as a provider, thus being able to have its life cycle controlled by our application being able to be injectable into other objects.
The rest of the class is just pure javascript, we keep our posts in an object called posts indexed by its id, everytime we create a new object we increment an variable called currentId that we use to set a post id. To return a list or a single post is just  as simple as it could be. Bear in mind that because we are keeping the posts in memory, every time we restart the application we
we´ll lose its values.
Moving foward, it is time to change our PostsController to have our new service injected and use it to retrieve/create posts instead of just hardcoded data.

**bloggering/src/posts/posts.controller.ts**

	import { Controller, Get, Param, Post, HttpCode, Body } from  '@nestjs/common';
	import { Post  as  BlogPost } from  './post.entity';
	import { PostsService } from  './posts.service';

	@Controller('posts')
	export  class  PostsController {
		constructor(private  readonly  postsService: PostsService) { }

		@Get()
		findAll(): BlogPost[] {
			return  this.postsService.getPosts();
		}

		@Get(':id')
		find(@Param('id') postId: number): BlogPost {
			return  this.postsService.getSinglePost(postId);
		}

		@Post()
		@HttpCode(201)
		create(@Body() newPost: BlogPost) {
			return  this.postsService.createPost(newPost.title, "me", newPost.content);
		}
	}
We created a constructor to the PostsController receiving our *PostsService* dependency. Because the *PostsService* has an @Injectable decorator and we set it as a PostsModule provider, NestJS knows how to resolve this dependency correctly.
The rest of the changes are pretty basic, we just removed the hardcoded data, from the methods and use our service to serve our data. In the create method we removed the HtpStatus(204), otherwise we would not be able to return the id of the recently created post.
If you want you can remove the *PostsService* from provider array in the *PostsModules* to see an dependency resolve error.
![NestJs scaffold structure](/images/create-nestjs-app/05-resolving-dependency-error.png)

By the time, if you are using start:dev (watch command), just save the files and the application will restart and reload automatically. Otherwise you will have to start the application again.
Lets test what we have done.

	PS D:\projects\bloggering> npm run start:dev

Create a post:
Request

	POST http://localhost:3000/api/posts/ HTTP/1.1
	Content-Type: application/json
	{
	    "title": "Working with NestJS",
	    "content": "Post content etc...etc...etc..."
	}
As response we shoud receive our new created id.

	HTTP/1.1 201 Created
	X-Powered-By: Express
	Content-Type: application/json; charset=utf-8
	ETag: W/"8-h5EdGu1QmHe4OkjsU292jNzSLfE"
	Date: Sat, 16 May 2020 22:00:26 GMT
	Connection: keep-alive
	{"id":1}
Now we can use this id, to retrieve this specific post with a GET

	GET http://localhost:3000/api/posts/1 HTTP/1.1
As a result the response should be something like that.

	HTTP/1.1 200 OK
	X-Powered-By: Express
	Content-Type: application/json; charset=utf-8
	ETag: W/"3e-6fB7zFoU1v8tbDih37C0yfN9O+Y"
	Date: Sat, 16 May 2020 22:06:10 GMT
	Connection: keep-alive

	{"author":"me","title":"Last","content":"dasdasdasdas","id":1}

You can create as many posts as you want, and retrieve all or just a specific one. They will remain until you restart you application.
Until now, our post creation does not have any validation, which means that we are allowing anyone who calls our api to create posts with empty titles or contents. Luckly for us, we can use two packages called **class-validator** and **class-transformer** . We can use a builtIn NestJS pipe called ValidationPipe to make validation as easily as include an decorator into the entity's property.

	PS C:\gems\blog-projects\bloggering> npm install class-validator --save
	PS C:\gems\blog-projects\bloggering> npm install class-transformer --save

Next, configure our app to use the *ValidationPipe* as a global pipe. This will apply the validations to all our endpoints. In the ***main.ts*** file add the following line inside the *boostrap* function.

**bloggering/src/main.ts**

	app.useGlobalPipes(new  ValidationPipe());
Now we can use a ton of decorators to validate our properties. Check them [here](https://github.com/typestack/class-validator#validation-decorators)
Add a @IsNotEmpty() decorator to our post content and title.
**bloggering/src/posts/posts.entity.ts**

	...
	@IsNotEmpty()
	title: string;
	@IsNotEmpty()
	content: string;

Lets try to create an post with an empty title and content now. Look how we receive a HttpStatus of 400 (Bad Request error) with a detailed information of the error.
![NestJs scaffold structure](/images/create-nestjs-app/06-validation-post-create.png)


##Apenas para lembrar
Ao usar o elastic search no docker compose, optamos por discovery.type=single-node para efeito de desenvolvimento. Ao acessar o elasticsearch ter noção da diferença entre acessar o localhost:9200 da máquina local e acessar via rede por outro container. Nesse caso usar o nome do container, ex: es01:9200