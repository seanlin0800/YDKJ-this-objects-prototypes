## Simpler Design

In addition to OLOO providing ostensibly simpler (and more flexible!) code, behavior delegation as a pattern can actually lead to simpler code architecture. Let's examine one last example that illustrates how OLOO simplifies your overall design.

The scenario we'll examine is two controller objects, one for handling the login form of a web page, and another for actually handling the authentication (communication) with the server.

We'll need a utility helper for making the Ajax communication to the server. We'll use jQuery (though any framework would do fine), since it handles not only the Ajax for us, but it returns a promise-like answer so that we can listen for the response in our calling code with `.then(..)`.

**Note:** We don't cover Promises here, but we will cover them in a future title of the *"You Don't Know JS"* series.

Following the typical class design pattern, we'll break up the task into base functionality in a class called `Controller`, and then we'll derive two child classes, `LoginController` and `AuthController`, which both inherit from `Controller` and specialize some of those base behaviors.

```js
// Parent class
function Controller() {
	this.errors = [];
}
Controller.prototype.showDialog = function(title,msg) {
	// display title & message to user in dialog
};
Controller.prototype.success = function(msg) {
	this.showDialog( "Success", msg );
};
Controller.prototype.failure = function(err) {
	this.errors.push( err );
	this.showDialog( "Error", err );
};
```

```js
// Child class
function LoginController() {
	Controller.call( this );
}
// Link child class to parent
LoginController.prototype = Object.create( Controller.prototype );
LoginController.prototype.getUser = function() {
	return document.getElementById( "login_username" ).value;
};
LoginController.prototype.getPassword = function() {
	return document.getElementById( "login_password" ).value;
};
LoginController.prototype.validateEntry = function(user,pw) {
	user = user || this.getUser();
	pw = pw || this.getPassword();

	if (!(user && pw)) {
		return this.failure( "Please enter a username & password!" );
	}
	else if (pw.length < 5) {
		return this.failure( "Password must be 5+ characters!" );
	}

	// got here? validated!
	return true;
};
// Override to extend base `failure()`
LoginController.prototype.failure = function(err) {
	// "super" call
	Controller.prototype.failure.call( this, "Login invalid: " + err );
};
```

```js
// Child class
function AuthController(login) {
	Controller.call( this );
	// in addition to inheritance, we also need composition
	this.login = login;
}
// Link child class to parent
AuthController.prototype = Object.create( Controller.prototype );
AuthController.prototype.server = function(url,data) {
	return $.ajax( {
		url: url,
		data: data
	} );
};
AuthController.prototype.checkAuth = function() {
	var user = this.login.getUser();
	var pw = this.login.getPassword();

	if (this.login.validateEntry( user, pw )) {
		this.server( "/check-auth",{
			user: user,
			pw: pw
		} )
		.then( this.success.bind( this ) )
		.fail( this.failure.bind( this ) );
	}
};
// Override to extend base `success()`
AuthController.prototype.success = function() {
	// "super" call
	Controller.prototype.success.call( this, "Authenticated!" );
};
// Override to extend base `failure()`
AuthController.prototype.failure = function(err) {
	// "super" call
	Controller.prototype.failure.call( this, "Auth Failed: " + err );
};
```

```js
var auth = new AuthController(
	// in addition to inheritance, we also need composition
	new LoginController()
);
auth.checkAuth();
```

We have base behaviors that all controllers share, which are `success(..)`, `failure(..)` and `showDialog(..)`. Our child classes `LoginController` and `AuthController` override `failure(..)` and `success(..)` to augment the default base class behavior. Also note that `AuthController` needs an instance of `LoginController` to interact with the login form, so that becomes a member data property.

The other thing to mention is that we chose some *composition* to sprinkle in on top of the inheritance. `AuthController` needs to know about `LoginController`, so we instantiate it (`new LoginController()`) and keep a class member property called `this.login` to reference it, so that `AuthController` can invoke behavior on `LoginController`.

**Note:** There *might* have been a slight temptation to make `AuthController` inherit from `LoginController`, or vice versa, such that we had *virtual composition* through the inheritance chain. But this is a strongly clear example of what's wrong with class inheritance as *the* model for the problem domain, because neither `AuthController` nor `LoginController` are specializing base behavior of the other, so inheritance between them makes little sense except if classes are your only design pattern. Instead, we layered in some simple *composition* and now they can cooperate, while still both benefiting from the inheritance from the parent base `Controller`.

If you're familiar with class-oriented (OO) design, this should all look pretty familiar and natural.

### De-class-ified

But, **do we really need to model this problem** with a parent `Controller` class, two child classes, **and some composition**? Is there a way to take advantage of OLOO-style behavior delegation and have a *much* simpler design? **Yes!**

```js
var LoginController = {
	errors: [],
	getUser: function() {
		return document.getElementById( "login_username" ).value;
	},
	getPassword: function() {
		return document.getElementById( "login_password" ).value;
	},
	validateEntry: function(user,pw) {
		user = user || this.getUser();
		pw = pw || this.getPassword();

		if (!(user && pw)) {
			return this.failure( "Please enter a username & password!" );
		}
		else if (pw.length < 5) {
			return this.failure( "Password must be 5+ characters!" );
		}

		// got here? validated!
		return true;
	},
	showDialog: function(title,msg) {
		// display success message to user in dialog
	},
	failure: function(err) {
		this.errors.push( err );
		this.showDialog( "Error", "Login invalid: " + err );
	}
};
```

```js
// Link `AuthController` to delegate to `LoginController`
var AuthController = Object.create( LoginController );

AuthController.errors = [];
AuthController.checkAuth = function() {
	var user = this.getUser();
	var pw = this.getPassword();

	if (this.validateEntry( user, pw )) {
		this.server( "/check-auth",{
			user: user,
			pw: pw
		} )
		.then( this.accepted.bind( this ) )
		.fail( this.rejected.bind( this ) );
	}
};
AuthController.server = function(url,data) {
	return $.ajax( {
		url: url,
		data: data
	} );
};
AuthController.accepted = function() {
	this.showDialog( "Success", "Authenticated!" )
};
AuthController.rejected = function(err) {
	this.failure( "Auth Failed: " + err );
};
```

Since `AuthController` is just an object (so is `LoginController`), we don't need to instantiate (like `new AuthController()`) to perform our task. All we need to do is:

```js
AuthController.checkAuth();
```

Of course, with OLOO, if you do need to create one or more additional objects in the delegation chain, that's easy, and still doesn't require anything like class instantiation:

```js
var controller1 = Object.create( AuthController );
var controller2 = Object.create( AuthController );
```

With behavior delegation, `AuthController` and `LoginController` are **just objects**, *horizontal* peers of each other, and are not arranged or related as parents and children in class-orientation. We somewhat arbitrarily chose to have `AuthController` delegate to `LoginController` -- it would have been just as valid for the delegation to go the reverse direction.

The main takeaway from this second code listing is that we only have two entities (`LoginController` and `AuthController`), **not three** as before.

We didn't need a base `Controller` class to "share" behavior between the two, because delegation is a powerful enough mechanism to give us the functionality we need. We also, as noted before, don't need to instantiate our classes to work with them, because there are no classes, **just the objects themselves.** Furthermore, there's no need for *composition* as delegation gives the two objects the ability to cooperate *differentially* as needed.

Lastly, we avoided the polymorphism pitfalls of class-oriented design by not having the names `success(..)` and `failure(..)` be the same on both objects, which would have required ugly explicit pseudopolymorphism. Instead, we called them `accepted()` and `rejected(..)` on `AuthController` -- slightly more descriptive names for their specific tasks.

**Bottom line**: we end up with the same capability, but a (significantly) simpler design. That's the power of OLOO-style code and the power of the *behavior delegation* design pattern.

