---
layout: post
title: Unit testing ExpressJS REST API
---

Recently, I had the opportunity to ramp up on NodeJS. We were building a web application with fairly simple architecture, NodeJS + ExpressJS based REST API on top of a SQL Azure database and an Angular + Bing maps based single page application as the client.

I wanted to share our approach to unit testing the api and get some feedback from the community. Following is a list of Node packages we used to build and test api:

* grunt
* mocha
* sinon
* chai
* supertest
* istanbul
* grunt-mocha-test
* grunt-mocha-istanbul

For the purpose of this discussion, I modified the default [express generator](http://expressjs.com/starter/generator.html) template and included above npm packages. I updated the default users route to use a mock storage service. Code in this blog is hosted @[expressjs-template](https://github.com/rajashekhark/expressjs-template), feel free to fork or download the template.

Next we will look at the code for each of the three modules, i.e., storage service, users route and unit test for the users route.

*Mock storage service:*
{% highlight js %}
var Q = require('q');
var users = [
{id: 1, name:'user1'},
{id: 2, name:'user2'},
{id: 3, name:'user3'}
];

var getStoreConnection = Q.fbind(function(){
  var deferred = Q.defer();
  deferred.resolve({});
  return deferred.promise.delay(1000);
});

var listAllUsers = Q.fbind(function(){
  var deferred = Q.defer();
  storage.getStoreConnection()
  .then(function(connection){
    deferred.resolve(users);
  })
  .fail(function(error){
    deferred.reject(error);
  });
  return deferred.promise.delay(1000);
});

var storage = {
  getStoreConnection: getStoreConnection,
  listAllUsers: listAllUsers
};

module.exports = storage;
{% endhighlight %}
*Users route has a GET method that calls the storage service:*
{% highlight js %}
var express = require('express');
var storage = require('../services/storage');
var router = express.Router();
var Q = require('q');

var getUsers = function(request, response){
  storage.listAllUsers()
  .then(function(users){
    response.send(users);
  })
  .fail(function(err){
    response.status(500).send('failed to list all users');
  });
};

router.get('/', getUsers);
module.exports = router;
{% endhighlight %}
*Writing unit tests for the users route using the aforementioned frameworks is fairly trivial:*
{% highlight js %}
var express = require('express');
var request = require('supertest');
var sinon = require('sinon');
var should = require('chai').should();
var Q = require('q');
var users = require('../../routes/users');
var storage = require('../../services/storage');
var app = require('../../app');

describe('users api', function (){

  var testUsers = [
  {id:1, name:'testUser1'},
  {id:2, name:'testUser2'}
  ];

  describe('GET /', function (){

    beforeEach(function(){
      sinon.stub(storage, 'getStoreConnection')
      .returns(Q.resolve({}));
      sinon.stub(storage, 'listAllUsers')
      .returns(Q.resolve(testUsers));
    });

    afterEach(function(){
      storage.getStoreConnection.restore();
      storage.listAllUsers.restore();
    });

    it('should respond with status 200 on success.', function (done){

      request(app)
      .get('/users')
      .end(function (err, res){
        res.status.should.equal(200);
        res.body.length.should.equal(2);
        if (err){
          return done(err);
        }
        done();
      });

    });

    it('should respond with status 500 on error.', function (done){

      storage.listAllUsers.restore();
      sinon.stub(storage, 'listAllUsers')
      .throws(new Error('error occured'));

      request(app)
      .get('/users')
      .end(function (err, res){
        res.status.should.equal(500);
        if (err){
          return done(err);
        }
        done();
      });
    });

  });
});
{% endhighlight %}
We used mocha framework's *beforeEach* and *afterEach* hooks to stub the dependencies and induce default behavior. Then depending on the test we customize the stub behavior, example, we configure the stub to throw an error to test for responses on failure.

Lastly we used Istanbul to pull coverage reports. Both unit tests and coverage reports are configured as grunt tasks. Images below show the output from the coverage task, which also executes the unit tests.

![grunt runcoverage output]({{site.url}}/images/expressjs_template_runcoverage.png)

Above combination of frameworks are working very well for us. I would love to hear experiences from others in the community.
