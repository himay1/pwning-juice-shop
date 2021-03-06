# Codebase 101

Jumping head first into any foreign codebase can cause a little
headache. This section is there to help you find your way through the
code of OWASP Juice Shop. On its top level the Juice Shop codebase is
mainly separated into a client and a server tier, the latter with an
underlying lightweight database and file system as storage.

## Client Tier

OWASP Juice Shop uses v1.5 of the popular
[AngularJS](https://angularjs.org/) framework as the core of its
client-side. Thanks to the [Bootstrap](http://getbootstrap.com/) CSS
framework, the UI is responsive letting it adapt nicely to different
screen sizes. Both frameworks work very well together thanks to the
[UI Bootstrap](https://angular-ui.github.io/bootstrap/) library that
provides components for Bootstrap that are explicitly written for
AngularJS. The various icons used throughout the frontend are from the
vast [Font Awesome 5](https://fontawesome.com/) collection.

![Client tier focus](img/architecture-client.png)

### Services

> AngularJS services are substitutable objects that are wired together
> using dependency injection (DI). You can use services to organize and
> share code across your app.[^1]

The client-side AngularJS services reside in the `app/js/services`
folder. Each service file handles all RESTful HTTP calls to the Node.js
backend for a specific domain entity or functional aspect of the
application.

![AngularJS Services folder](img/servicesFolder.png)

Service functions must **always** use `$q.defer()` to wrap the return
value of the `$http` backend call into a `promise`. If the backend call
was successful, the promise will be resolved. In case of an error, the
promise will be rejected.

The following code snippet shows how all services in the OWASP Juice
Shop client are structured using the example of `FeedbackService`. It
wraps the `/api/Feedback` API which offers a `GET`, `POST` and `DELETE`
endpoint to find, create and delete `Feedback` of users:

```javascript
angular.module('juiceShop').factory('FeedbackService', ['$http', '$q', function ($http, $q) {
  'use strict'

  var host = '/api/Feedbacks'

  function find (params) {
    var feedbacks = $q.defer()
    $http.get(host + '/', {params: params}).success(function (data) {
      feedbacks.resolve(data.data)
    }).error(function (err) {
      feedbacks.reject(err)
    })
    return feedbacks.promise
  }

  function save (params) {
    var createdFeedback = $q.defer()
    $http.post(host + '/', params).success(function (data) {
      createdFeedback.resolve(data.data)
    }).error(function (err) {
      createdFeedback.reject(err)
    })
    return createdFeedback.promise
  }

  function del (id) {
    var deletedFeedback = $q.defer()
    $http.delete(host + '/' + id).success(function (data) {
      deletedFeedback.resolve(data.data)
    }).error(function (err) {
      deletedFeedback.reject(err)
    })
    return deletedFeedback.promise
  }

  return {
    find: find,
    save: save,
    del: del
  }
}])
```

:rotating_light: Unit tests for all services can be found in the
`test/client/services` folder. They are
[Jasmine 3](https://jasmine.github.io) specifications which are executed
by the [Karma](https://karma-runner.github.io) test runner.

### Controllers

> In AngularJS, a Controller is defined by a JavaScript constructor
> function that is used to augment the AngularJS Scope.[^2]

The AngularJS controllers reside in the `app/js/controllers` folder.
Each controller file handles is responsible for one screen or functional
aspect of the application.

![AngularJS Controllers folder](img/controllersFolder.png)

Controllers must **always** go through one or more [Services](#services)
when communicating with the application backend. Furthermore, they will
rely on their own `$scope` and are forbidden to pollute the `$rootScope`
unnecessarily.

The code snippet below shows the `ContactController` which handles the
_Contact Us_ screen and uses three different services to fulfill its
tasks:
* `UserService` to retrieve data about the currently logged in user (if
  applicable) via the `whoAmi()` function
* `CaptchaService` to retrieve a new CAPTCHA for the user to solve via
  the `getCaptcha()` function
* `FeedbackService` to eventually `save()` the user feedback

:point_up: As a universal rule for the entire Juice Shop codebase,
unnecessary code duplication should be avoided by using well-named
helper functions. This is demonstrated by the very simple
`getNewCaptcha()` function in the code snippet below. Helper functions
should always be located as close to the calling code as possible and
**never** be put into the _global scope_ unnecessarily as this can cause
unexpected side effects.

```javascript
angular.module('juiceShop').controller('ContactController', [
  '$scope',
  'FeedbackService',
  'UserService',
  'CaptchaService',
  function ($scope, feedbackService, userService, captchaService) {
    'use strict'

    userService.whoAmI().then(function (data) {
      $scope.feedback = {}
      $scope.feedback.UserId = data.id
      $scope.userEmail = data.email || 'anonymous'
    })

    function getNewCaptcha () {
      captchaService.getCaptcha().then(function (data) {
        $scope.captcha = data.captcha
        $scope.captchaId = data.captchaId
      })
    }
    getNewCaptcha()

    $scope.save = function () {
      $scope.feedback.captchaId = $scope.captchaId
      feedbackService.save($scope.feedback).then(function (savedFeedback) {
        $scope.error = null
        $scope.confirmation = 'Thank you for your feedback' + (savedFeedback.rating === 5 ? ' and your 5-star rating!' : '.')
        $scope.feedback = {}
        getNewCaptcha()
        $scope.form.$setPristine()
      }).catch(function (error) {
        $scope.error = error
        $scope.confirmation = null
        $scope.feedback = {}
        $scope.form.$setPristine()
      })
    }
  }])
```

:rotating_light: Unit tests for all controllers can be found in the
`test/client/controllers` folder. Like the
[service unit tests](#services) they are written in
[Jasmine 3](https://jasmine.github.io) and run on
[Karma](https://karma-runner.github.io).

### Views

> In AngularJS, templates are written with HTML that contains
> AngularJS-specific elements and attributes. AngularJS combines the
> template with information from the model and controller to render the
> dynamic view that a user sees in the browser.[^3]

Each screen within the application is defined in a HTML view template in
the folder `app/views`. The views are written as HTML5 using
[Bootstrap](http://getbootstrap.com/) for styling and responsiveness.
Furthermore most views incorporate some
[UI Bootstrap](https://angular-ui.github.io/bootstrap/) component
specifically for AngularJS and several icons from the
[Font Awesome 5](https://fontawesome.com/) collection.

![AngularJS Views folder](img/viewsFolder.png)

The following code snippet shows the `Contact.html` view which, together
with the previously shown `ContractController.js` represent the _Contact
Us_ screen.

```html
<div class="row">
    <section class="col-md-6 col-md-offset-3 col-sm-8 col-sm-offset-2">
        <h3 class="page-header page-header-sm" translate="TITLE_CONTACT"></h3>

        <div>

            <form role="form" name="form" novalidate>
                <input type="text" id="userId" ng-model="feedback.UserId" ng-hide="true"/>
                <div class="alert-info" ng-show="confirmation && !form.$dirty">
                    <p>{{confirmation}}</p>
                </div>
                <div class="alert-danger" ng-show="error && !form.$dirty">
                    <p>{{error}}</p>
                </div>
                <div class="alert-danger" ng-show="form.$invalid && form.$dirty">
                    <p ng-show="form.feedbackComment.$error.required && form.feedbackComment.$dirty" translate="MANDATORY_COMMENT"></p>
                    <p ng-show="form.feedbackComment.$error.maxlength && form.feedbackComment.$dirty" translate="INVALID_COMMENT_LENGTH" translate-value-length="1-160"></p>
                    <p ng-show="form.feedbackRating.$error.required && form.feedbackRating.$dirty" translate="MANDATORY_RATING"></p>
                    <p ng-show="form.feedbackCaptcha.$error.required && form.feedbackCaptcha.$dirty" translate="MANDATORY_CAPTCHA"></p>
                </div>

                <div class="container-fluid well">

                        <div class="row">
                            <div class="form-group">
                                <label translate="LABEL_AUTHOR"></label>
                                <label class="form-control input-sm">{{userEmail}}</label>
                            </div>
                        </div>
                        <div class="row">
                            <div class="form-group">
                                <label for="feedbackComment" translate="LABEL_COMMENT"></label>
                                <textarea class="form-control input-sm" id="feedbackComment" name="feedbackComment" ng-model="feedback.comment" required ng-maxlength="160"></textarea>
                            </div>
                        </div>
                        <div class="row">
                            <div class="form-group">
                                <label translate="LABEL_RATING"></label>
                                <span uib-rating max="5" name="feedbackRating" ng-model="feedback.rating" required></span>
                            </div>
                        </div>
                        <div class="row">
                            <div class="form-group">
                                <label translate="LABEL_CAPTCHA"></label>&nbsp;
                                <code id="captcha">{{captcha}}</code>&nbsp;<label>?</label>
                                <input class="form-control input-sm" name="feedbackCaptcha" ng-model="feedback.captcha" required/>
                            </div>
                        </div>
                        <div class="row">
                                <div class="form-group">
                                    <button type="submit" id="submitButton" class="btn btn-primary" ng-disabled="form.$invalid" ng-click="save()"><i class="fas fa-paper-plane fa-lg"></i> <span translate="BTN_SUBMIT"></span></button>
                                </div>
                        </div>
                </div>
             </form>

        </div>

    </section>
</div>
```

### Index page template

The `app/index.template.html` file is the entry point the Juice Shop web
client. It loads all required CSS stylesheets and JavaScript libraries
and includes the application's own client-side code. It also defines the
surrounding elements of the application, such as the
* navigation bar and menu items
* challenge-related pop-ups
* cookie consent banner
* _Fork me on GitHub_ ribbon.

For developers it is important to only make changes in the
`index.template.html` file but **never** in the `index.html` which will
be generated from the template during server startup. This intermediary
step is necessary for the visual
[Customization](../part1/customization.md) of the application.

### Internationalization

All static texts in the user interface are fully internationalized using
the `angular-translate` module. Texts coming from the server (e.g.
product descriptions or server error messages) are always in English.

No hard-coded texts are allowed in any of the [Views](#views) or
[Controllers](#controllers). Instead, property keys have to be defined
and are usually specified in a `translate` attribute which can be placed
in most HTML tags and will later be resolved by the `$translateProvider`
service. You might have noticed several of these `translate` attributes
in the `Contact.html` code snippet from the [Views](#views) section.

The different translations are maintained in JSON files in the
`/app/i18n` folder. The only file that should be touched by developers
is the `en.json` file for the original English texts. New properties are
exclusively added here. When pushing the `develop` branch to GitHub, the
online translation provider will pick up changes in `en.json` and adapt
all other language files accordingly. All this happens behind the scenes
in a distinct branch `l10n_develop` which will be manually merged back
into `develop` on a regular basis.

To learn about the actual translation process please refer to the
chapter [Helping with translations](translation.md).

### Client-side code minification

All client side code (except the `index.html`) is first _uglified_ (for
[security by obscurity](https://en.wikipedia.org/wiki/Security_through_obscurity))
and then _minified_ (for initial load time reduction) during the build
process (launched with `npm install`) of the application. This creates
an `app/dist/juice-shop.min.js` file, which is included by the
`index.html` to load all application-specific client-side code.

If you want to quickly test client-side code changes, it can be
cumbersome to launch `npm install` over and over again. Instead you can
simply trigger the minification to generate the `juice-shop.min.js` file
with

```bash
grunt minify
```

and then refresh your browser with `F5` to test your changes. This will
require grunt being installed globally on your system, so if above
command fails for you, please run `npm install -g grunt-cli` once to
install this useful task runner. From then on, `grunt minify` should
work.

## Server Tier

The backend of OWASP Juice Shop is a [Node.js](https://nodejs.org)
application based on the [Express](http://expressjs.com) web framework.

![Server tier focus](img/architecture-server.png)

:information_source: On the server side all JavaScript code must be
compliant to javascript (ES6) syntax.

### Routes

> Routing refers to determining how an application responds to a client
> request to a particular endpoint, which is a URI (or path) and a
> specific HTTP request method (GET, POST, and so on).
> 
> Each route can have one or more handler functions, which are executed
> when the route is matched.[^4]

Routes are defined via the the [Express](http://expressjs.com) framework
and can be handled by any of the following middlewares:

* An [automatically generated API endpoint](#generated-api-endpoints)
  for one of the exposed tables from the application's
  [Data model](#data-model)
* A [hand-written middleware](#hand-written-middleware) which
  encapsulates some business or technical responsibility
* Some third-party middleware that fulfills a non-functional requirement
  such as
  * file serving (via `serve-index` and `serve-favicon`)
  * adding HTTP security headers (via `helmet` and `cors`)
  * extracting cookies from HTTP requests (via `cookie-parser`)
  * writing access logs (via `morgan`)
  * catching unhandled exceptions and presenting a default error screen
    (via `errorhandler`)

:rotating_light: Integration tests for all routes can be found in the
`test/api` folder alongside all other API endpoint tests, from where
[Frisby.js](https://www.frisbyjs.com/)/[Jest](https://facebook.github.io/jest/)
assert the functionality of the entire backend on HTTP-request/response
level.

#### Generated API endpoints

Juice Shop uses the [Epilogue](https://github.com/dchester/epilogue)
middleware to automatically create REST endpoints for most of its
Sequelize models. For e.g. the `User` model the generated endpoints are:

* `/api/Users` accepting
  * `GET` requests to retrieve all (or a filtered list of) user records
  * and `POST` requests to create a new user record
* `/api/Users/{id}` accepting
  * `GET` requests to retrieve a single user record by its database ID
  * `PATCH` requests to update a user record
  * `DELETE` requests to delete a user record

Apart from the `User` model also the `Product`, `Feedback`,
`BasketItem`, `Challenge`, `Complaint`, `Recycle`, `SecurityQuestion`
and `SecurityAnswer` models are exposed in this fashion.

Not all HTTP verbs are accepted by every endpoint. Furthermore, some
endpoints are protected against anonymous access and can only be used by
an authenticated user. This is described later in section
[Access control on routes](#access-control-on-routes).

```javascript
epilogue.initialize({
  app,
  sequelize: models.sequelize
})

const autoModels = ['User', 'Product', 'Feedback',
'BasketItem', 'Challenge', 'Complaint', 'Recycle',
'SecurityQuestion', 'SecurityAnswer']

for (const modelName of autoModels) {
  const resource = epilogue.resource({
    model: models[modelName],
    endpoints: [`/api/${modelName}s`, `/api/${modelName}s/:id`]
  })

  // fix the api difference between epilogue and previously
  // used sequlize-restful
  resource.all.send.before((req, res, context) => {
    context.instance = {
      status: 'success',
      data: context.instance
    }
    return context.continue
  })
}
```

#### Hand-written middleware

The business functionality in the application backend is separated into
tightly scoped middleware components which are placed in the `routes`
folder.

![Express routes folder](img/routesFolder.png)

These middleware components are directly mapped to
[Express](http://expressjs.com) routes.

Each middleware exposes a single function which encapsulates their
responsibility. For example, the `angular.js` middleware delivers the
`index.html` page to the client:

```javascript
const path = require('path')
const utils = require('../lib/utils')

module.exports = function serveAngularClient () {
  return ({url}, res, next) => {
    if (!utils.startsWith(url, '/api') && !utils.startsWith(url, '/rest')) {
      res.sendFile(path.resolve(__dirname, '../app/index.html'))
    } else {
      next(new Error('Unexpected path: ' + url))
    }
  }
}
```

If a hand-written middleware is involved in a hacking challenge, it must
assess on its own if the challenge has been solved. For example, in the
`basket.js` middleware where successfully accessing another user's
shopping basket is verified:

```javascript
const utils = require('../lib/utils')
const insecurity = require('../lib/insecurity')
const models = require('../models/index')
const challenges = require('../data/datacache').challenges

module.exports = function retrieveBasket () {
  return (req, res, next) => {
    const id = req.params.id
    models.Basket.find({ where: { id }, include: [ { model: models.Product, paranoid: false } ] })
      .then(basket => {
        if (utils.notSolved(challenges.basketChallenge)) {
          const user = insecurity.authenticatedUsers.from(req)
          if (user && id && id !== 'undefined' && user.bid != id) {
            utils.solve(challenges.basketChallenge)
          }
        }
        res.json(utils.queryResultToJson(basket))
      }).catch(error => {
        next(error)
      })
  }
}
```

The only middleware deviating from above specification is `verify.js`.
It contains no business functionality. Instead of one function it
exposes several named functions on challenge verification for
[Generated API endpoints](#generated-api-endpoints), for example:

```
app.post('/api/Feedbacks', verify.forgedFeedbackChallenge())
app.post('/api/Feedbacks', verify.captchaBypassChallenge())
```

The same applied for any challenges on top of third-party middleware,
for example:

```
app.use(verify.errorHandlingChallenge())
app.use(errorhandler())
```

Similar to the [Generated API endpoints](#generated-api-endpoints), not
all hand-written endpoints can be used anonymously. The upcoming section
[Access control on routes](#access-control-on-routes) explains the
available authorization checks.

:rotating_light: Unit tests for hand-written routes can be found in the
`test/server` folder. These tests are written using the
[Chai](http://chaijs.com/) assertion library in conjunction with the
[Mocha](https://mochajs.org/) test framework.

#### Access control on routes

For both the generated and hand-written middleware access can be
retricted on the corresponding routes by adding `insecurity.denyAll()`
or `insecurity.isAuthorized()` as an extra middleware. Examples for
denying all access to certain HTTP verbs for the `SecurityQuestion` and
`SecurityAnswer` models:

```javascript
/* SecurityQuestions: Only GET list of questions allowed. */
app.post('/api/SecurityQuestions', insecurity.denyAll())
app.use('/api/SecurityQuestions/:id', insecurity.denyAll())

/* SecurityAnswers: Only POST of answer allowed. */
app.get('/api/SecurityAnswers', insecurity.denyAll())
app.use('/api/SecurityAnswers/:id', insecurity.denyAll())
```

The following snippet show the authorization settings for the `User`
model which allows only `POST` to anonymous users (for registration) and
requires to be logged-in for retrieving the list of users or individual
user records. Deleting users is completely forbidden:

```javascript
app.get('/api/Users', insecurity.isAuthorized())
app.route('/api/Users/:id')
  .get(insecurity.isAuthorized())
  .put(insecurity.denyAll()) // Updating users is forbidden to make the password change challenge harder
  .delete(insecurity.denyAll()) // Deleting users is forbidden entirely to keep login challenges solvable
```

### Custom libraries

Two important and widely used custom libraries reside in the `lib`
folder, one containing useful utilities (`lib/utils.js`) and the other
encapsulating many of the broken security features (`lib/insecurity.js`)
of the application.

#### Useful utilities

The main responsibility of the `utils.js` module is setting challenges
as solved and sending associated notifications, optionally including a
CTF flag code. It can also retrieve any challenge by its name and check
if a passed challenge is not yet solved, to avoid unnecessary (and
sometimes expensive) repetitive solving of the same challenge.

```javascript
exports.solve = function (challenge, isRestore) {
  const self = this
  challenge.solved = true
  challenge.save().then(solvedChallenge => {
    solvedChallenge.description = entities.decode(sanitizeHtml(solvedChallenge.description, {
      allowedTags: [],
      allowedAttributes: []
    }))
    console.log(colors.green('Solved') + ' challenge ' + colors.cyan(solvedChallenge.name) + ' (' + solvedChallenge.description + ')')
    self.sendNotification(solvedChallenge, isRestore)
  })
}

exports.sendNotification = function (challenge, isRestore) {
  if (!this.notSolved(challenge)) {
    const flag = this.ctfFlag(challenge.name)
    const notification = {
      name: challenge.name,
      challenge: challenge.name + ' (' + challenge.description + ')',
      flag: flag,
      hidden: !config.get('application.showChallengeSolvedNotifications'),
      isRestore: isRestore
    }
    notifications.push(notification)
    if (global.io) {
      global.io.emit('challenge solved', notification)
    }
  }
}
```

It also offers some basic `String` and `Date` utilities along with data
(un-)wrapper functions and a method for the synchronous file download
used during [Customization](../part1/customization.md).

#### Insecurity features

The `insecurity.js` module offers all security-relevant utilities of the
application, but of course mostly in some broken or flawed way:

* Hashing functions borh weak (`hash()`) and relatively strong
  (`hmac()`)
* [Route](#routes) authorization via JWT with `denyAll()` and
  `isAuthorized()` (see
  [Access control on routes](#access-control-on-routes)) and
  corresponding grant of permission for a users with `authorize()`
* HTML sanitization by exposing a (vulnerable) external library as
  function `sanitizeHtml()`
* Keeping a bi-directional map of users with their current
  authentication token (JWT) in `authenticatedUsers`
* Coupon code creation and verification functions `generateCoupon()` and
  `discountFromCoupon()`
* A whitelist of allowed redirect URLs and a corresponding check
  function `isRedirectAllowed()`
* CAPTCHA verification via `verifyCaptcha()` which compares the user's
  answer against the requested CAPTCHA from the database

## Storage Tier

[SQLite](https://www.sqlite.org) and
[MarsDB](https://github.com/c58/marsdb) form the backbone of the Juice
Shop, as an e-commerce application without storage for its product,
customer and associated data would not be very realistic. The Juice Shop
uses light-weight implementations on the database layer to keep it
runnable as a single "all-inclusive" server which
[can be deployed in various ways](../part1/running.md#run-options) with
ease.

![DB tier focus](img/architecture-database.png)

### Database

For the main database of the Juice Shop the file-based
[SQLite](https://www.sqlite.org) engine is used. It does not require a
separate server but is accessed directly from `data/juiceshop.sqlite` on
the file system of the Node.js server. For ease of use and more
flexibility the relational mapping framework
[Sequelize](http://docs.sequelizejs.com) is used to actually access the
data through a querying API. Sometime plain SQL is used as well, and of
course in an unsafe way that allows [Injection](../part2/injection.md).

#### Data model

The relational data model of the Juice Shop is very straightforward. It
features the following tables:

* `Users` which contains all registered users (i.e. potential customers)
  of the web shop.
* The table `SecurityQuestions` contains a fixed number of security
  questions a user has to choose from during registration. The provided
  answer is stored in the table `SecurityAnswers`.
* The `Products` table contains the products available in the shop
  including price data.
* When logging in every user receives a shopping basket represented by a
  row in the `Baskets` table. When putting products into the basket this
  is reflected by entries in `BasketItems` linking a product to a basket
  together with a quantity.
* Users can interact further with the shop by
  * giving feedback which is stored in the `Feedbacks` table
  * complaining about recent orders which creates entries in the
    `Complaints` table
  * asking for fruit-pressing leftovers to be collected for recycled via
    the `Recycles` table.
* The table `Captchas` stores all generated CAPTCHA questions and
  answers for comparison with the users response.
* The `Challenges` table would not be part of the data model of a normal
  e-commerce application, but for simplicities sake it is kept in the
  same schema. This table stores all hacking challenges that the OWASP
  Juice Shop offers and persists if the user already solved them or not.

![ERM Diagram](img/erm-diagram.png)

### Non-relational database

Not all data of the Juice Shop resides in a relational schema. The
product `reviews` are stored in a non-relational in-memory
[MarsDB](https://github.com/c58/marsdb) instance. An example user
`reviews` entry might look like the following inside MarsDB:

```
{"message":"One of my favorites!","author":"admin@juice-sh.op","product":1,"_id":"PaZjAKKMaxWieSF65"}
```

All interaction with MarsDB happens via the MongoDB query syntax.

### Populating the databases

The OWASP Juice Shop comes with a `data/datacreator.js` module that is
automatically executed on every server start after the SQLite file and
in-memory MarsDB have been cleared. It populates all tables with some
initial data which makes the application usable out-of-the-box:

```javascript
module.exports = async () => {
  const creators = [
    createUsers,
    createChallenges,
    createRandomFakeUsers,
    createProducts,
    createBaskets,
    createBasketItems,
    createFeedback,
    createComplaints,
    createRecycles,
    createSecurityQuestions,
    createSecurityAnswers
  ]

  for (const creator of creators) {
    await creator()
  }
}
```

For the `Users` and `Challenges` tables the rows to be inserted are
defined via YAML files in the `data/static` folder. As the contents of
the `Products` table and the non-relational `reviews` collection
[can be customized](../part1/customization.md), it is populated based on
the active configuration file. By default this is `config/default.yml`).

The data in the `Feedbacks`, `SecurityQuestions`, `SecurityAnswers`,
`Basket`, `BasketItem`, `Complaints` and `Recycles` tables is statically
defined within the `datacreator.js` script. They are so simple that a
YAML declaration file seemed like overkill.

The `Captchas` table remains empty on startup, as it will dynamically
generate a new CAPTCHA every time the _Contact us_ page is visited.

### File system

The folder `ftp` contains some files which are directly accessible. When
a user completes a purchase, an order confirmation PDF is generated and
placed into this folder. Other than that the `ftp` folder is also used
to deliver the shop's terms of use to interested customers.

#### Uploaded complaint files

The _File complaint_ page contains a file upload field to attach one of
the previously mentioned order confirmation PDFs. While these are really
uploaded to the server, they are _not written_ to the file system but
discarded for security reasons: Publicly hosted Juice Shop instances are
not supposed to be abused a malware distribution sites or file shares.

## End-to-end tests

> As applications grow in size and complexity, it becomes unrealistic to
> rely on manual testing to verify the correctness of new features,
> catch bugs and notice regressions. Unit tests are the first line of
> defense for catching bugs, but sometimes issues come up with
> integration between components which can't be captured in a unit test.
> End-to-end tests are made to find these problems.[^5]

The folder `test/e2e` contains an extensive suite of end-to-end tests
which **automatically solves _every_ challenge** in the Juice Shop
application. Whenever a new challenge is added, a corresponding
end-to-end test needs to be included, to prove that it can be exploited.

It is quite an impressive sight to see how
{{book.juiceShopNumberOfChallenges}} hacking challenges are solved
without any human interaction in a few minutes. The e2e tests constantly
jump back and forth between attacked pages and the
[Score Board](../part1/challenges.md#the-score-board) letting you watch
as the difficulty stars and progress bar slowly fill and ever more green
"solved"-badges appear. There is a
[video recording of this on YouTube](https://www.youtube.com/watch?v=oiFUdZlS7zI)
for the 7.0.0 release of the Juice Shop.

[![OWASP Juice Shop 7.0.0 - Protractor test suite](img/protractor-youtube.png)](https://www.youtube.com/watch?v=oiFUdZlS7zI)

These tests are written and executed with
[Protractor](https://www.protractortest.org) which uses
[Selenium WebDriver](https://www.seleniumhq.org/projects/webdriver/)
under the hood.

[^1]: https://docs.angularjs.org/guide/services

[^2]: https://docs.angularjs.org/guide/controller

[^3]: https://docs.angularjs.org/guide/templates

[^4]: http://expressjs.com/en/starter/basic-routing.html

[^5]: https://docs.angularjs.org/guide/e2e-testing

