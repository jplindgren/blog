---
date: 2020-04-10T10:58:08-03:00
description: "Creating a simple but complete backend blog application with NestJS, Postgres and ElasticSearch"
featured_image: "/images/nestjs-logo-small.svg"
tags: [ "Javascript", "Nestjs", "OAuth"]
series: ["Complete NestJS"]
title: "Creating a NestJS application (Part I)"
summary: "Part I of the series 'Creating a NestJs application'. In this part we are going to have an overview of NestJs, scaffold the basic structure of our application, add a basic controller, use a service and implement authentication using Google OAuth."
weight: 99
slug: part1
---
BE AWARE THAT THIS IS A WORK IN PROGRESS... I UPLOADED AS PUBLISHED TO HELP PEOPLE AND GET FEEDBACKS.
IF YOU WANT TO HELP ME WITH FEEDBACKS, YOU CAN SEND AN EMAIL TO joaopozo@gmail.com

Part I
- [What is NestJS](#what-is-nestjs)
- [Creating a Project](#creating-a-project)
- [Adding Controllers](#add-controllers)
- [Adding Services](#add-services)
- [Adding Authentication](#add-google-auth)

[Part II]({{< relref "creating-nestjs-application-part2.md" >}})
- [Add Database Persistence]({{< relref "creating-nestjs-application-part2.md#add-persistence-with-typeorm" >}})
- [Configuration]({{< relref "creating-nestjs-application-part2.md#add-configuration-service" >}})
- [Docker and Docker compose]({{< relref "creating-nestjs-application-part2.md#add-docker-and-docker-compose" >}})
- [Add a post subscriber and index content with elastic search]({{< relref "creating-nestjs-application-part2.md#add-elastic-search" >}})

During this pandemic with the free time, I and some friends wanted to do a side project to improve our skills and know new technologies and frameworks. I´ll not focus on what would be this project, but after a few talks we decided to do it, using react native and NodeJs in the backend.
It turns out that we did not take this side project out of paper for several reasons, conciliate schedules, different objectives, and lack of agreement as to the direction in which the project should go, among other things. During our discussions one of my friends presented this framework called NestJS as an option to use in the backend. Although we halt the project, I decided to give it a shot to this framework, and started a little web application on my own just to learn it.

This article describes in detail the steps I took in creating a "fake" blog API. I start by explaining what NestJS is, how you can set it up, add controllers, persist data, and dockerize it. Be warned that this is just a SAMPLE PROJECT, and while some parts are kept very simple or even lacking, in other aspects you will see some OVERENGINEER because I wanted to discover how things work in NestJS.

## What is NestJS
As you can see [here](https://docs.nestjs.com/), the documentation claims NestJS to be a framework for building efficient, scalable NodeJs applications. It uses progressive JavaScript and combines OOP (Object Oriented Programming), FP (Functional Programming), and FRP (Functional Reactive Programming). Putting the ~~fancy words~~ buzzwords aside, it creates an abstraction on top of [express](https://expressjs.com/) (or [fastify](https://www.fastify.io/)) providing an opinionated application **architecture**. I kind of like opinionated frameworks, although you can customize things, and do it your way, they kind of strongly guide you to use certain tools and code using their style as you can see looking into their documentation. Because of that, create something from scratch is super fast and you can jump in a new project without spending much time thinking how you are going to organize things, "it is not so easy" to break the rules and create messy code using it. A nice thing to have especially for newcomers to the framework. Also they use typescript which is very positive to me (I confess that I had preconception, but after using it for some time, I fell in love with it) and seems to have a decent CLI.
One thing that can be a positive point for some, is that it looks a lot like [Angular.js](https://angularjs.org/). As Angular is not my background, it was indifferent to me, but I think that it should be very straightforward for angular developers to jump into NestJS.

## Creating the project

Well, I decided to create a ~~complex and innovative~~ old but gold sample project! An API for a blog, where we will be able to retrieve, list, create, update, and publish posts! Exciting no!? =]
Jokes apart, I think we can use a lot of tools creating a sample project like this, and to be honest, it was the first idea that came to my mind to create an article.
I am using Windows, Windows Terminal, and Node v14.2.0.

**Let's install the CLI (Command-line interface) package and scaffold a new project.**

	PS D:\projects> npm i -g @nestjs/cli
	PS D:\projects> nest new bloggering
	PS D:\projects\bloggering> cd bloggering

We just installed the CLI globally and use it to scaffold our new project with the default arguments (you can use args to change language, package manager, etc). The CLI already create a folder for us with the basic project structure. The entry point of the application is the ***main.ts*** where we create our application and define the port it will listen.
If you want this is the place where you can set a global prefix for your project. Let's add for instance a global prefix for our API.
**bloggering/src/main.ts**

```typescript {linenos=inline,linenostart=5}
const app = await NestFactory.create(AppModule);
app.setGlobalPrefix('api');
```

Opening the folder in an IDE its structure should look like.
![NestJS scaffold structure](/images/create-nestjs-app/01-project-structure-after-post-module.png)

We can run the application now to check it.

    PS D:\projects\bloggering> npm run start

You can use fiddler, postman, curl or even the browser and navigate to http://localhost:3000 to see a "hello world* message.

## Add controllers

Let's get into the action and add our fist controller. We´ll create a posts controller that will be responsible to handle all the basic operations over a Post, like create, edit, list, get, etc.
First create a `PostsModule` where we´ll use to organize everything related to posts into it. Modules are the way NestJS uses to organize your application dependencies keeping each feature segregated and following SOLID principles. Every NestJS has at least one module, the root module. Let's use the CLI to create one for us using the command generate with the argument ***module*** (mo).
Also let's create a `PostsController` as well using the command generate controller (co).

    PS D:\projects\bloggering> nest generate mo posts
    PS D:\projects\bloggering> nest generate co posts

![NestJS scaffold structure](/images/create-nestjs-app/02-project-structure-after-create.png)
Look that nest created a folder called **/posts** with a `posts.controller.ts` and `posts.module.ts` for us. Besides that, it also updated our `app.module.ts` to import our new **PostsModule**.
In the **src/posts** folder create a file `post.entity.ts` that will contain our post entity.
After that Inside our new `posts.controller.ts` we´ll create a couple of endpoints to deal with requests to /posts routes. In the end we should have these two files.

**bloggering/src/posts/post.entity.ts**
```typescript {linenos=inline}
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
```
**bloggering/src/posts/posts.controller.ts**
```typescript {linenos=inline}
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
```
Until now everything is pretty straightforward. Just basic endpoints with some fake data. Later we are going to create DTOs to avoid expose inner objects to the external world and save and retrieve real posts. By the time the only points worth of notice are:

- I had to import the Post entity as PostEntity. I had to do that,
because the name collides with the Post method decorator from NestJS.
- In the *find* method we defined a route param prefixing with ':' like **:id**. We can access its value decorating the argument received by the method with "@Param('**id**'). Of course the param defined in the route and the decorator has to match.
- In the *create* method we set the HTTP return status as 204 (no content). And although we did not use it yet, we decorated our argument newPost with the @Body. This means that the content of the body of the HTTP request will be mapped to this field.

Run your application and try to access our news endpoints at the post controller. http://localhost:3000/api/posts. If you don´t know you can use start:dev to run in watch mode. Using this we can run our application and reloads our application automatically every time we modify and save a file.

	PS D:\projects\bloggering> npm run start:dev

NestJS show us every route mapped in the build output on the terminal.
![NestJS scaffold structure](/images/create-nestjs-app/03-terminal-with-routes.png)
Both Post and Get verbs are already responding
![NestJS scaffold structure](/images/create-nestjs-app/04-post-api-calls.png)
In the next section we are going to create a post service and start to deal with real posts!

## Add services

Currently we have our controllers just creating hardcoded data to return from its endpoints. Now we are going to create a `PostsService` to save/retrieve dynamically created post entities. But, they will not be saved in the database yet, instead, we´ll save them in memory for a while.
After creating this service, we will need to include it as a provider in the `PostsModule`, this will tell NestJS injector the scope of this dependency and help it control its instantiation and encapsulation to other parts of the system. Then we can inject the service into the `PostsController` and use it to retrieve and create our posts.
To automatically update the `PostsModule` dependencies, let's use the CLI to generate our service. As before, the command pattern is the same, we just have to change the schematics name to service.

	PS D:\projects\bloggering> nest generate service posts

NestJS will automatically create a `posts.service.ts` file inside the posts folder for us, and update the provider of our PostsModule.
Later we are adding [TypeORM](https://typeorm.io/#/) to persist our posts in a database, but for now, the application will handle the posts in memory. Lets code our PostsService class.

**bloggering/src/posts/posts.service.ts**
```typescript {linenos=inline}
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
```

Pretty simple right? We imported the post entity (by the way, because of the typescript awesomeness, in some IDEs like [VSCode](https://code.visualstudio.com/) if it's not imported you can simply press ctrl+. and choose the option to import Post entity from module *post.entity*), then we decorate our class with the @Injectable() marking it as a provider, thus being able to have its life cycle controlled by our application being able to be injectable into other objects.
The rest of the class is just pure javascript, we keep our posts in an object called posts indexed by its id, every time we create a new object we increment a variable called currentId that we use to set a post id. To return a list or a single post is just as simple as it could be. Bear in mind that because we are keeping the posts in memory, every time we restart the application we´ll lose its values.
Moving forward, it is time to change our PostsController to have our new service injected and use it to retrieve/create posts instead of just hardcoded data.

**bloggering/src/posts/posts.controller.ts**
```typescript {linenos=inline}
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
```
We created a constructor to the `PostsController` receiving our `PostsService` dependency. Because the `PostsService` has an `@Injectable` decorator and we set it as a `PostsModule` provider, NestJS knows how to resolve this dependency correctly.
The rest of the changes are pretty basic, we just removed the hardcoded data, from the methods and use our service to serve our data. In the create method we removed the HtpStatus(204), otherwise, we would not be able to return the id of the recently created post.
If you want you can remove the *PostsService* from provider array in the *PostsModules* to see a dependency resolve error.
![NestJS scaffold structure](/images/create-nestjs-app/05-resolving-dependency-error.png)

By the time, if you are using start:dev (watch command), just save the files and the application will restart and reload automatically. Otherwise you will have to start the application again.
Let's test what we have done.

	PS D:\projects\bloggering> npm run start:dev

Create a post using the request below:
**Request**

	POST http://localhost:3000/api/posts/ HTTP/1.1
	Content-Type: application/json
	{
		"title": "Working with NestJS",
		"content": "Post content etc...etc...etc..."
	}

As a response we should receive our newly created id.

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

You can create as many posts as you want, and retrieve all or just a specific one. They will remain until you restart your application.
Until now, our post creation does not have any validation, which means that we are allowing anyone who calls our API to create posts with empty titles or contents. Luckily for us, we can use two libraries called **class-validator** and **class-transformer** to add validations to our application. Using a builtIn NestJS pipe called `ValidationPipe` to make validation as easily as include a decorator into the entity's property.

	PS C:\gems\blog-projects\bloggering> npm install class-validator --save
	PS C:\gems\blog-projects\bloggering> npm install class-transformer --save

Next, configure our app to use the `ValidationPipe` as a global pipe. This will apply the validations to all our endpoints. In the `main.ts` file add the following line inside the `bootstrap` function.

**bloggering/src/main.ts**

```typescript {linenos=inline, linenostart=7}
app.useGlobalPipes(new  ValidationPipe());
```

Now we can use a ton of decorators to validate our properties. Check them [here](https://github.com/typestack/class-validator#validation-decorators).
Add a `@IsNotEmpty()` decorator to our post content and title.

**bloggering/src/posts/posts.entity.ts**
```typescript {linenos=false,hl_lines=[2, 4]}
...
@IsNotEmpty()
title: string;
@IsNotEmpty()
content: string;
```
Try to create a post with an empty title and content now and see how we receive a HttpStatus of 400 (Bad Request error) with detailed information on the error.
![NestJS scaffold structure](/images/create-nestjs-app/06-validation-post-create.png)

## Add Google Auth
By now, we are using a hardcoded author when creating a new post. In this section lets add a **User** entity to allow a person to authenticate and use our app. Also we are going to add [Guards](https://docs.nestjs.com/guards) to some of our endpoints. A Guard is a decorator class that has the responsibility to define if and decorated endpoint will handle or not the request based on certain conditions.
To accomplish that, we will use a popular and recommended by NestJS library called **passport**.  This library allows you to create "strategies" to do different types of authentication. I will bypass the local authentication using a username/password because there is a pretty straightforward tutorial in the NestJS documentation that you can check [here](https://docs.nestjs.com/techniques/authentication). Instead, we will use OAuth2 to authenticate using a google account. I did not try, but probably you can use the same idea to implement other providers like Facebook, Microsoft, etc.
This is an overview of how our auth flow will look like.
![NestJS scaffold structure](/images/create-nestjs-app/07-oauth2-google-sequence.png)
Usually, we would have a frontend as the client consuming our API, receiving the auth token, and saving it to append in each subsequent requests. But for our example, we will consume the API directly and append the authorization header manually.

 -  a client (in our case for test purposes, the browser, or some API test tool like insomnia or postman) calling the **/api/auth/google** route which will redirect to the Google login page.
 - the user gives consent to login and google calls a callback endpoint in our application with the user data.
 - our application will do some logic to log in or create the user if needed and create a JWT auth token that will be returned to the client.
 - the client will call our protected APIs passing the JWT token in the authorization header in each subsequent request.

First let's install some dependencies.

	PS C:\gems\blog-projects\bloggering> npm install @nestjs/passport --save
	PS C:\gems\blog-projects\bloggering> npm install passport --save
	PS C:\gems\blog-projects\bloggering> npm install passport-google-oauth20 --save

Create an `auth module`, an `auth controller`, and an `auth service`.

    PS D:\projects\bloggering> nest generate mo auth
    PS D:\projects\bloggering> nest generate co auth
	PS D:\projects\bloggering> nest generate service auth


After that, let's configure our application API in google to get our credentials. Go to [https://console.developers.google.com](https://console.developers.google.com) and register the bloggering application.
Create a new project, select it, and enable the Google+ API by clicking on 'enable APIs and services' and searching for the Google+ API. Then go to 'credentials' and select the 'OAuth client ID' option when trying to create credentials.

Choose 'Web application' as the application type and continue. Add  `http://localhost:3000`  under the Authorized Javascript origins and add  `http://localhost:3000/api/auth/google/callback`  under Authorized redirect URIs. Google will redirect the user information to our callback URI when a user successfully logs in. Save and replace the  `clientID`  and  `clientSecret` because we will need for the google strategy passport.
![NestJS scaffold structure](/images/create-nestjs-app/08-oauth2-google-console-config.png)

Ok, finally we've set all dependencies, configurations and we are ready to code! We are going to start creating a folder inside the `/auth` folder called `strategies`. Inside that folder create a `google.passport.strategy.ts` file. In the future we can implement other strategies here as well, like local strategy.

**bloggering/src/auth/strategies/google.passport.strategy.ts**
```typescript {linenos=inline,hl_lines=["10-12",14]}
import { Injectable } from  "@nestjs/common";
import { PassportStrategy } from  "@nestjs/passport";
import { Strategy } from  "passport-google-oauth20";

@Injectable()
export  class GooglePassportStrategy extends  PassportStrategy(Strategy, 'google')
{
	constructor() {
		super({
			clientID:'ClIENT_ID', // <- Replace this with your client id
			clientSecret: 'CLIENT_SECRET', // <- Replace this with your client secret
			callbackURL: 'http://localhost:3000/api/auth/google/callback',
			passReqToCallback:  true,
			scope: ['profile', 'email']
		})
	}

	async  validate(request: any, accessToken: string, refreshToken: string, profile, done: Function) {
		try {
			const  jwt = "placeholderJWT";
			const  user = { jwt };
			done(null, user);
		}
		catch (err) {
			done(err, false);
		}
	}
}
```
This class extends `PassportStrategy` calling its constructor with our credentials from google. Copy them to the clientId and clientSecret parameters of `super()`. Never put them on your code! Later we´ll remove and store them in environments variables. Also we passed our callback URL, and the scope of the information we need to retrieve from google.
	Our `validate` function will handle a successful login on google, when it calls our callback endpoint. Later we are applying some registration logic and create a JWT token, but by now just return a hardcoded string to serve us for test purposes. You can put a `console.log` or debug to see the content of the `profile` object that google sent to us.
>REMEMBER to not commit these credentials to your source control

Import the `GooglePassportStrategy` as a provider in the `AuthModule`.

**bloggering/src/auth/auth.module.ts**
```typescript {linenos=inline,hl_lines=[9]}
import { Module } from  '@nestjs/common';
import { AuthController } from  './auth.controller';
import { AuthService } from  './auth.service';
import { GooglePassportStrategy } from  './strategies/google.passport.strategy';

@Module({
	imports: [],
	controllers: [AuthController],
	providers: [AuthService, GooglePassportStrategy],
})
export  class  AuthModule { }
```
Now to be able to test what we have done we have to create endpoints to handle the requests from the client and from google! Remember the callback endpoint that google will call?
Let's implement our `AuthController`

**bloggering/src/auth/auth.controller.ts**
```typescript {linenos=inline,hl_lines=[7, 13]}
import { Controller, Get, UseGuards, Res, Req } from  '@nestjs/common';
import { AuthGuard } from  '@nestjs/passport';

@Controller('auth')
export  class  AuthController {
	@Get('google')
	@UseGuards(AuthGuard('google'))
	oauth2GoogleLogin() {
		// initiates the Google OAuth2 login flow
	}

	@Get('google/callback')
	@UseGuards(AuthGuard('google'))
	oauth2GoogleLoginCallback(@Req() req, @Res() res) {
		const  jwt: string = req.user.jwt;
		if (jwt)
			res.status(200).json({ jwtToken:  jwt });
		else
			res.status(400).json({ errorMessage:  'Authentication failed' });
	}
}
```
The first endpoint will be responsible to trigger our OAuth2 flow and start the communication with google. Pay attention that we use the AuthGuard passing the same name we used in the on our `GooglePassportStrategy` class. Under the hood, the passport will find our strategy and call google passing our credentials.
The second endpoint will be our callback that google will call when the user gives his consent to the login. This because of the AuthGuard, passport will find our google passport strategy and call the validate method. This validate method if you remember it, calls a `done(null, user)`callback. This function is responsible to populate our request with the user object.

Check whether your application works by going to your browser and typing http://localhost:3000/api/auth/google. If everything works correctly, you should be redirected to a google login web page, and after fill the information you should be redirected again to our application with the token in the response.

![NestJS scaffold structure](/images/create-nestjs-app/09-oauth2-google-signin.png)

Next we are going to get rid of this hardcoded token and create a real JWT token. After installing NestJS JWT dependency we´ll implement the `AuthService` to create a user and generate our authorization token based on the info we received from google.  For a while our user will be temporary, but as soon as we have a real persistence in the database we are going to register a real user in the correct table.

Create a `User` entity and implement an `AuthService`

**bloggering/src/user/user.entity.ts**
```typescript {linenos=inline}
export  class  User {
		id: string;
		thirdPartyId: string;
		name: string;
		email: string;
		isActive: boolean;
		createdAt!: Date;
}
```
Next install our next JWT dependency

	PS D:\projects\bloggering> npm i @nestjs/jwt --save
We will need to register the `JwtModule` in our `AuthModule` to be able to inject a configured `JwtService` in our `AuthService`. Again, DO NOT LET SENSITIVE INFORMATION LIKE YOUR JWT SECRET IN YOUR CODE. We well see how to remove it later.

**bloggering/src/auth/auth.module.ts**
```typescript {linenos=inline,hl_lines=["8-11"]}
import { Module } from  '@nestjs/common';
import { AuthController } from  './auth.controller';
import { AuthService } from  './auth.service';
import { GooglePassportStrategy } from  './strategies/google.passport.strategy';
import { JwtModule } from  '@nestjs/jwt';

@Module({
	imports: [JwtModule.register({
		secret:  'hard!to-guess_secret',
		signOptions: { expiresIn:  '1200s' }
	})],
	controllers: [AuthController],
	providers: [AuthService, GooglePassportStrategy],
})

export  class  AuthModule { }
```
In our case this secret is used both to sign in and to verify the token. Because we will have only one API using this secret is easy to just store it in an environment variable, or other secret files.

**bloggering/src/auth/auth.service.ts**
```typescript {linenos=inline,hl_lines=[7, "19-23"]}
import { Injectable, InternalServerErrorException } from  '@nestjs/common';
import { User } from  'src/users/user.entity';
import { JwtService } from  '@nestjs/jwt';

@Injectable()
export  class  AuthService {
	constructor(private  jwtService: JwtService) {
	}

	async  validateOAuthLogin(email: string, name: string, thirdPartyId: string, provider: string): Promise<string> {
		try {
			const  fakeuser = new  User();
			fakeuser.id = new  Date().getTime().toString(); //fake id;
			fakeuser.email = email;
			fakeuser.isActive = true;
			fakeuser.name = name;
			fakeuser.thirdPartyId = thirdPartyId;

			return this.jwtService.sign({
				id:  fakeuser.id,
				entity:  fakeuser,
				provider
			});
		}
		catch (err) {
			throw  new  InternalServerErrorException('validateOAuthLogin', err.message);
		}
	}
}
```
Currently, we do not have any logic yet. Instead our method just creates a fake user with the real data from google and create a signed [jwt token](https://jwt.io/introduction/) that expires 1200 seconds with the data we want it to contains.
This should be a generic service and you have to try to keep all concrete dependencies to a specific provider out of this class. Following this guideline should be easy to just implement passports for other providers and just call this same class.

Additionally change `GooglePassportStrategy` to inject the `AuthService` into the  constructor and use it in the `validate` function.

**bloggering/src/auth/strategies/google.passport.strategy.ts**
```typescript {linenos=inline,hl_lines=[3, 11]}
@Injectable()
export  class  GooglePassportStrategy  extends  PassportStrategy(Strategy, 'google')
	constructor(private  readonly  authService: AuthService){
		...
	}
	async  validate(request: any, accessToken: string, refreshToken: string, profile, done: Function) {
		try {
			const  email = profile.emails
				.filter(x  =>  x.verified)
				.map(x  =>  x.value)[0];
			const  jwt = await  this.authService.validateOAuthLogin(email, profile.displayName, profile.id, 'google');
			done(null, { jwt });
		}
		catch (err) {
			done(err, false);
		}
	}
}
```

We are almost finishing our authentication feature! We are still missing a @Guard to protect endpoints by requiring a valid JWT to be present on the request. **Passport** can help us here again with a [passport-jwt](https://github.com/mikenicholson/passport-jwt) strategy for securing RESTful endpoints with JSON Web Token.
Install passport-jwt dependency.

	PS C:\gems\blog-projects\bloggering> npm install passport-jwt --save

Create a new strategy called `JwtStrategy` inside our `/auth/strategies` folder

**bloggering/src/auth/strategies/jwt.strategy.ts**
```typescript {linenos=inline}
import { ExtractJwt, Strategy } from  'passport-jwt';
import { PassportStrategy } from  '@nestjs/passport';
import { Injectable } from  '@nestjs/common';

@Injectable()
export  class  JwtStrategy  extends  PassportStrategy(Strategy) {
	constructor() {
		super({
			jwtFromRequest:  ExtractJwt.fromAuthHeaderAsBearerToken(),
			ignoreExpiration:  false,
			secretOrKey:  'hard!to-guess_secret', //same as you put in the JwtModule.register on auth.module.ts.
		});
	}

	async  validate(payload: any) {
		return { userId:  payload.id, name:  payload.entity.name, email:  payload.entity.email };
	}
}
```

This class is responsible to extract the JWT token from the request authorization header, check if it is valid, verifies JWT's signature, it is not expired. If we have some custom verifications like check claims, if the user is still active, etc we can add here. We could have a dependency on an `UserService` and load our user from the database.
In the return of the validate method we just map our token payload to a simple object.
This class will be used the same way as our `google.passport.strategy`, we will decorate

Do not forget to add our `JwtStrategy` as a provider in the `AuthModule`.

**bloggering/src/auth/auth.module.ts**
```typescript {linenos=inline}
...
providers: [AuthService, GooglePassportStrategy, JwtStrategy],
...
```

Last but not least, let's update our `posts.controller` creation method to have an AuthGuard to check if the user is authenticated or not, and change the hardcoded author to be our user instead.

**bloggering/src/posts/posts.controller.ts**
```typescript {linenos=inline, hl_lines=[6]}
...
import { Controller, Get, Param, Post, HttpCode, Body, UseGuards, Req } from  '@nestjs/common';
import { AuthGuard } from  '@nestjs/passport/dist';
...
@Post()
@UseGuards(AuthGuard('jwt'))
create(@Body() newPost: BlogPost, @Req() req) {
	const  user = req['user'];
	return  this.postsService.createPost(newPost.title, user.name, newPost.content)
}
...
```
To test whether our endpoint is protected or not, make a call without passing an authorization header and check if it returns 401 as a result.
Now to test if everything is working, get the token you received from google and make a call to create a post passing it as an authorization header.

	POST http://localhost:3000/api/posts/ HTTP/1.1
	Content-Type: application/json
	Authorization: Bearer <jwt_token_here>

	{
	    "title": "My First Post",
	     "content": "Check Post"
	}

We should receive an HTTP status 201 as a result.
Make another call to get this single created post and check how our user is populated.

Wow! That was a lot of work! In part II we are going to add TypeORM to persist our data into a real database, extract our sensitive data to an environment file and expose it to our application using a ConfigService, add Docker, and DockerCompose to manage our services, and index our post content using elastic search.