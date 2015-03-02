---
layout: post
title: "RACCommand's immediateEnabled property"
date: 2015-03-01 21:11:59 -0300
comments: true
categories:
  - objective-c
  - ios
  - reactivecocoa
  - frp
  - mvvm
  - bdd
---
For the last couple months I've been doing a lot of [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) for different iOS projects and let me tell you; it is pretty fucking awesome!!!. I know it's been around for [a couple years](https://github.com/ReactiveCocoa/ReactiveCocoa/releases/tag/v0.5.0) ... yeah yeah [Swift](https://developer.apple.com/library/mac/documentation/Swift/Conceptual/Swift_Programming_Language/index.html) is the big thing now, Objective-C is dead and the [Swift version](https://github.com/ReactiveCocoa/ReactiveCocoa/pull/1382) of ReactiveCocoa is around the corner (maybe).

*I don't actually think Objective-C is dead and I am probably still going to use it, at least until the libraries get a little more mature and XCode stops crashing 10 times per day.*

This post is NOT about how happy I feel using **ReactiveCocoa** or how it feels so right using it to implement the [MVVM](http://en.wikipedia.org/wiki/Model_View_ViewModel) pattern. This post is about something I learned about [RACCommand](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/ReactiveCocoa/RACCommand.h)'s internals.

As you may [know](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/Documentation/FrameworkOverview.md#commands) `RACCommand` is an abstraction that models a command, a user initiated action that may have some side effects. You can create a new `RACCommand` object using the `initWithEnabled:signalBlock:` initializer method. This method receives a signal as the first parameter and a block that receives an input and returns a signal as the second parameter.

The block will be called every time the `execute:` method is invoked on the `RACCommand` object. The object that receives the block is the object that is passed to `execute:`. The signal returned by the block will be the signal returned by `execute:`.

The signal is used to decide whether the command is enabled or disabled. This is pretty useful because when you do something like `self.button.rac_command = self.viewModel.someCommand` the enabled property of the button is automatically changed when the command is enabled or disabled, avoiding all the boilerplate code to keep the button state synced.

Assuming we have the following interfaces
```objc
@interface SessionService : NSObject

@property (nonatomic, readonly) User * currentUser;

- (RACSignal *)loginWithUsername:(NSString *)username password:(NSString *)password;

- (RACSignal *)logout;

@end

@interface LoginViewModel : NSObject

@property (nonatomic, readonly, getter=isUsernameValid) BOOL usernameValid;
@property (nonatomic, readonly, getter=isPasswordValid) BOOL usernameValid;
@property (nonatomic, readonly) RACCommand * login;

@property (nonatomic) NSString * username;
@property (nonatomic) NSString * password;

@end
```

a possible implementation for the login command could be

```objc
@implementation LoginViewModel

- (instancetype)init {
  self = [super init];
  if (self) {
    _username = @"";
    _password = @"";

    RAC(self, usernameValid) = [RACObserve(self, username) map:^(NSString * username) {
      return @(username != nil && username.length > 5);
    }];
    RAC(self, passwordValid) = [RACObserve(self, password) map:^(NSString * password) {
      return @(password != nil && password.length > 4);
    }];

    RACSignal * loginEnabled = [RACSignal combineLatest:@[
      RACObserve(self, usernameValid),
      RACObserve(self, passwordValid)
    ] reduce:^(NSNumber * usernameValid, NSNumber * passwordValid) {
        return @(usernameValid.boolValue && passwordValue.boolValue);
    }];

    _login = [[RACCommand alloc] initWithEnabled:loginEnabled signalBlock:^(id sender) {
      return [SessionService loginWithUsername:self.username password:self.password];
    }];
  }
  return self;
}

@end
```

Based on this implementation, unless the username has 5 characters and the password has 4 characters the login command will not be enabled. Executing a disabled command (calling its `execute:` method) will result in a signal that will error with domain `RACCommandErrorDomain` and code `RACCommandErrorNotEnabled`.

Now lets analyze a different example of how to use a `RACCommand`. Lets take for instance [pagination](http://en.wikipedia.org/wiki/Pagination). Most of the apps nowadays have some kind of newsfeed or activity stream. A very simple implementation of this view could fetch all the required data and display it on a `UITableView`. This could work pretty well if the data that needs to be displayed is not really big. But if we are talking about something like the Twitter's newsfeed doing just one query to the backend service to display all the user's newsfeed could result in a [DOS](http://en.wikipedia.org/wiki/Denial-of-service_attack) or at least it would take a lot of time to answer. This is good situation to apply pagination.

We can implement a simple view model that knows how to display a paginated list and later we can bind that view model against a `UITableViewController`. We can call that view model `TableViewModel` and a naive implementation of that view model could be

```objc
typedef RACSignal * (^PagedFetcher)(NSUInteger);

@interface TableViewModel : NSObject

@property (nonatomic, copy) PagedFetcher fetcher;
@property (nonatomic, readonly) NSUInteger count;
@property (nonatomic, readonly) RACCommand * fetchNextPage;
@property (nonatomic, readonly) BOOL consumedAllPages;

- (instancetype)initWithFetcher:(PagedFetcher)fetcher;

- (id)objectAtIndexedSubscript:(NSUInteger)index;

@end

@interface TableViewModel ()

@property (nonatomic) NSMutableArray * data;
@property (nonatomic) NSUInteger nextPage;

@end

@implementation TableViewModel

@dynamic count;

- (instancetype)initWithFetcher:(PagedFetcher)fetcher {
  self = [super init];
  if (self) {
    _fetcher = fetcher;
    _nextPage = 0;
    _consumedAllPages = NO;
    _data = [NSMutableArray array];
    @weakify(self)
    _fetchNextPage = [[RACCommand alloc] initWithEnabled:RACObserve(self, consumedAllPages)
                                             signalBlock:^(id value) {
                                              @strongify(self)
                                              return [self performFetch];
                                             }];
  }
  return self;
}

- (NSUInteger)count {
  return data.count;
}

- (id)objectAtIndexedSubscript:(NSUInteger)index {
  return self.data[index];
}

#pragma mark - Private Methods

- (RACSignal *)performFetch {
  return [[self.fetcher(self.nextPage) map:^(NSArray * data) {
    self.consumedAllPages = data.count == 0;
    [self.data addObjectsFromArray:data];
    self.nextPage++;
    return data;
  }] replay];
}

@end
```
As you can see the implementation of `TableViewModel` is pretty simple, the important part is in the private `performFetch` method that is called inside the signal block associated with the `fetchNextPage` command. *We are using replay before returning the signal in `performFetch` to cache the result of the map and
avoid the execution of the side effects (to increase the `nextPage` counter) in case several subscriptions get created to this signal.*

Now that we have implemented `TableViewModel` it's time to test the it and in order to do that I use [Specta](https://github.com/specta/specta) + [Expecta](https://github.com/specta/expecta) matchers + [OCMockito](https://github.com/jonreid/OCMockito). *For the purpose of this blog post I am only going to show a reduced version of the `TableViewModelSpec`.*

The following spec asserts that after calling `fetchNextPage` the page counter gets increased. In this case we are calling `fetchNextPage` 3 times thus making the last value of `requestPage` equal to 2 (because we have requested page 0, 1 and 2). We only want to fetch a page after the previous page was successfully fetched. That is why we are using `concat:` because it will subscribe to the concatenated signal after the first signal has completed. `completionSignal` is a signal that will first fetch page 0 and after it's completed it will fetch page 1 and after it's completed it will fetch page 2 and then will complete. If any of the concatenated signals errors `completionSignal`, will error immediately.

```objc
SpecBegin(TableViewModel)

describe(@"#fetchNexPage", ^{

  __block TableViewModel * tableViewModel;
  __block NSUInteger requestPage;

  beforeEach(^{
    tableViewModel = [[TableViewModel alloc] initWithFetcher:^(NSUInteger page) {
      requestedPage = page;
      NSArray * data = @[mock([NSObject class]), mock([NSObject class])];
      return [RACSignal return:data];
    }];
  });

  context(@"when some pages have been fetched", ^{

    __block RACSignal * completionSignal;

    beforeEach(^{
        completionSignal = [[[tableViewModel.fetchNextPage execute:nil]
                             concat:[tableViewModel.fetchNextPage execute:nil]]
                             concat:[tableViewModel.fetchNextPage execute:nil]];
    });

    it(@"increases the page number", ^{ waitUntil(^(DoneCallback done) {
      [completionSignal subscribeCompleted:^{
        expect(requestPage).to.equal(2);
        done();
      }];
    });});

  });

});

SpecEnd
```

Unfortunately if you run the previous spec you will get the following error

    failed to invoke done() callback before timeout (10.000000 seconds)

meaning that for some reason the subscribed block never got executed and the only way that that could've happened is if one
of the concatenated signals has failed. To verify this theory we can `subscribeError:` instead of `subscribeCompleted`.
When I did this I realized that indeed the signal was sending an error and the error code was `RACCommandErrorNotEnabled`.
This is super weird because the `fetchNextPage` command is enabled/disabled based on the `consumedAllPages` property and
the only way this could be set to `NO` is if the fetcher's signal returns an empty array and that is impossible because
we are using a fake fetcher that always returns a non-empty array.

Digging a little bit inside the internals of `RACCommand` I realized that `execute:` does not actually use the given signal to
decide if the command can be executed or not. (Check [this](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/ReactiveCocoa/RACCommand.m#L222) line and also [this](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/ReactiveCocoa/RACCommand.m#L203) line). When the `execute:`
method is invoked, it first gets a value from `immediateEnabled` which is a combination of the provided enabled signal
and another signal which is basically based on `allowsConcurrentExecution`. `immediateEnabled` sends `YES` if
the enabled signal sends `YES` and if `allowsConcurrentExecution`
is `NO` (which is the default) the `executing` property must
be `NO`.

What is happening and causing the test to fail is that when `execute:` gets invoked for the second time the change on
the `executing` signal has not been propagated yet and although the first invocation of `execute:` has finished the
internal state of the `RACCommand` does not reflect that.

In a real-case scenario this is virtually impossible to happen. At least if the `RACCommand` is bound to an event
triggered by the user, because this would probably happen in two different run loops and by that time the change
on the `executing` property would be propagated.

Finally the easy fix to make the test pass is to add the following statement in the `beforeEach` block

    tableViewModel.fetchNextPage.allowsConcurrentExecution = YES;

Which I think is a valid trade-off to be made. What do you guys think? Do you have a better solution?
