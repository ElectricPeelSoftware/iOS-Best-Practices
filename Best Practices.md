# iOS Development Best Practices

This document is a description of the practices used in iOS development at [Electric Peel](http://www.electricpeelsoftware.com).

We make heavy use of [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) and the [MVVM](http://en.wikipedia.org/wiki/Model_View_ViewModel) pattern, so this document expects familiarity with both.

## Frameworks & Tools

- [Default Podfile](https://github.com/ElectricPeelSoftware/iOS-Best-Practices/blob/master/Podfile)

### 3rd Party

- [CocoaPods](http://cocoapods.org)
- [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)
- [AFNetworking](https://github.com/AFNetworking/AFNetworking)

### Ours

- [EPSReactiveTableViewController](https://github.com/ElectricPeelSoftware/EPSReactiveTableViewController)
- [EPSReactiveCollectionViewController](https://github.com/ElectricPeelSoftware/EPSReactiveCollectionViewController)
- [EPSStaticTableViewController](https://github.com/ElectricPeelSoftware/EPSStaticTableViewController)

## Auto Layout

Auto layout should be used for all layout, except for in subclasses of `UITableViewCell` which use the existing `textLabel` and/or `detailTextLabel` labels.

### View Controllers

In general, view controllers should not have any auto layout code—if layout code is necessary, it should be done in subclasses of `UIView`. The view controller’s main view should be created in `-loadView`. The `view` property should be overridden with the class of the view that will be used as the main view.

```objective-c
@interface EPSPersonViewController ()
@property (nonatomic) EPSPersonView *view;
@end

@implementation EPSPersonViewController
...
- (void)loadView {
  self.view = [[EPSPersonView alloc] initWithFrame:[[UIScreen mainScreen] bounds]];
}
@end
```

A view should have public properties for the labels, etc. that it contains, so they can be modified by the view controller.

### Views

Views shouldn't be given an explicit width or height using constraints. If the size of a view can't be determined by the constraints on it relative to its sibling views, or by the constraints it has on its subviews, the `-intrinsicContentSize` method of the view should be overridden.

## View Models

A view model should model the content of the view it will be used with. Any data that will be shown in the view should have a corresponding property in the view model. The `-init` method of a view controller should take the model object it’s adapting for display as a parameter. For example, for a view which displays information about a person (which cannot be edited) might have a view model with this interface:

```objective-c
@interface EPSPersonViewModel : NSObject

@property (readonly, nonatomic) NSString *fullName;
@property (readonly, nonatomic) NSString *emailAddress;
@property (readonly, nonatomic) NSString *birthDate;
@property (readonly, nonatomic) UIImage *avatarImage;

- (id)initWithPerson:(EPSPerson *)person;

@end
```

Since the view does not support editing this information, all the properties are `readonly`. Note that the user’s birth date is exposed as an `NSString`, not an `NSDate`. The view model is responsible for transforming information like an `NSDate` into a format suitable for presentation. Bindings for all the properties should be set up in the `-init` method:

```objective-c
- (id)initWithPerson:(EPSPerson *)person {
  self = [super init];
  if (self == nil) return nil;

  _person = person;

  RAC(self, fullName) = [RACSignal
    combineLatest:@[ RACObserve(person, firstName), RACObserve(person, lastName) ]
    reduce:^NSString *(NSString *firstName, NSString *lastName){
      return [NSString stringWithFormat:@"%@ %@", firstName, lastName];
    }];
  RAC(self, emailAddress) = RACObserve(person, emailAddress);
  RAC(self, birthDate) = [RACObserve(person, birthDate) map:^NSString *(NSDate *birthDate){
    return [NSDateFormatter localizedStringFromDate:birthDate dateStyle:NSDateFormatterLongStyle timeStyle:NSDateFormatterNoStyle];
  }];
  RAC(self, avatarImage) = [[[APIClient sharedClient] avatarImageForPerson:person] catch:^RACSignal *(NSError *error){
    return [RACSignal return:nil];
  }];

  return self;
}
```

(Not shown: Since the properties were declared as `readonly` in the interface, they need to be redeclared as `readwrite` in a class extension in the implementation file.)

Since `RAC` bindings will crash if a signal that’s bound to something sends an error (an operation that depends on the network, for example), any errors sent on those signals must be handled by using `-catch:`, as is done in the `avatarImage` binding in the example above.

Actions the user takes should be represented by `RACCommand` properties on the view model. For example, in a login view with username and password text fields, and a “Log In” button, the button’s action should be provided by a `RACCommand` property.

```objective-c
@property (nonatomic) NSString *username;
@property (nonatomic) NSString *password;

@property (readonly, nonatomic) RACCommand *logInCommand;
```

The `username` and `password` properties are both `readwrite` since the user can edit the username and password in text fields. `logInCommand` is `readonly`, is the command itself should not be modified by the view controller.

A command to log in a user should disable itself if the username and password are not valid (ie. have a length of 0). The command should be created in the `-init` method of the view controller, and assigned to the `logInCommand` property.

```objective-c
- (id)init {
  ...
  @weakify(self);

  RACSignal *enabled = [RACSignal
    combineLatest:@[ RACObserve(self, username), RACObserve(self, password) ]
    reduce:^NSNumber *(NSString *username, NSString *password){
      return @(username.length > 0 && password.length > 0);
    }];
  _logInCommand = [[RACCommand alloc] initWithEnabled:enabled signalBlock:^RACSignal *(id input) {
    @strongify(self);

    RACSignal *signal = ... // A signal that logs the user in using `self.username` and `self.password`
    return signal;
  }];
  ...
}
```

### Error Handling

Errors as a result of the execution of a `RACCommand` should be handled using command’s `errors` signal. If the view controller needs to deal with the error, it should subscribe to the `errors` signal of the command:

```objective-c
- (id)init {
  ...
  [self.viewModel.loginCommand.errors subscribeNext:^(NSError *error){
    // Do something with `error`
  }];
  ...
}
```

Errors that are not a result of a `RACCommand` (for example, an error loading the data for the view), should be made available via an `errors` property on the view model.


```objective-c
@property (readonly, nonatomic) RACSignal *errors;
```

*Give an example of how to create this signal.*

## View Controllers

XIBs and storyboards should not be used to create view controllers. `-init` or a corresponding method like `-initWithPerson:` should be used to initialize view controllers, not `-initWithNibName:bundle:`.

All view controllers should have a corresponding view model class, named the same as the view controller, but with “Controller” replaced by “Model”. For example, `EPSExampleViewController` and `EPSExampleViewModel`. The view model should be created in the view controller’s `init` method, and saved in a private property called `viewModel`.

```objective-c
@interface EPSExampleViewController ()
@property (nonatomic) EPSExampleViewModel *viewModel;
@end

@implementation EPSExampleViewController
- (id)init {
  self = [super init];
  if (self == nil) return nil;

  _viewModel = [EPSExampleViewModel new];

  return self;
}
...
@end
```

All business logic should be handled in view models. A view controller should only be responsible for presentation of data, and navigation to other view controllers. All bindings to the view model should be set up in `-viewDidLoad`.

```objective-c
- (void)viewDidLoad {
  [super viewDidLoad];

  // Read-only bindings
  RAC(self.titleLabel, text) = RACObserve(self.viewModel, title);
  RAC(self.avatarView, image) = RACObserve(self.viewModel, avatarImage);

  // Write-only bindings
  RAC(self.viewModel, password) = self.passwordField.rac_textSignal;

  // Two-way bindings
  RACChannelTerminal *viewModelTerminal = RACChannelTo(self.viewModel, description);
  RACChannelTerminal *textFieldTerminal = [self.descriptionField rac_newTextChannel];
  [viewModelTerminal subscribe:textFieldTerminal];
  [textFieldTerminal subscribe:viewModelTerminal];
}
```

### Table and Collection View Controllers

The contents of a table view or collection view should be based on an `NSArray` property of the view controller’s view model. If it should be possible to add and remove objects from the list, the view model should provide public methods for that.

```objective-c
@interface EPSExampleViewModel : NSObject

@property (nonatomic) NSArray *people; // An array of EPSPersonViewModel objects

- (void)addPerson:(EPSPerson *)person;
- (void)removePerson:(EPSPerson *)person;

@end
```

Note that the `people` array in this example isn’t an array of `EPSPerson` objects—it contains `EPSPersonViewModel` objects. Cells should be given a view model to use, not direct access to a model object.

#### Cells

Like view controllers, cells should have a corresponding view model class. There should be a property for that view model:

```objective-c

@interface EPSPersonCell : UITableViewCell

@property (nonatomic) EPSPersonViewModel *personViewModel;

@end
```

When the property is set, the cell should update itself to reflect the new view model, and start observing the view model’s properties for changes. This should all be done with bindings in the `-initWithStyle:reuseIdentifier:` method:

```objective-c
- (id)initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier {
  self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
  if (self == nil) return nil;

  // Set up labels, image views, etc.

  RAC(self.nameLabel, text) = [[RACObserve(self, viewModel)
    map:^RACSignal *(EPSPersonViewModel *viewModel){
      return RACObserve(viewModel, fullName);
    }]
    switchToLatest];
  ...

  return self;
}
```

(Setting up the bindings is more complicated than in view controllers, because the cell needs to observe its view model property, as well as the properties of the current view model. Use `-map:` to return a `RACObserve` signal, and `-switchToLatest` to ensure that only the current view model’s changes are observed.)

#### Patterns

##### Table Views

- Static table view controllers should subclass [EPSStaticTableViewController](https://github.com/ElectricPeelSoftware/EPSStaticTableViewController).
- Dynamic table view controllers with only one section should subclass [EPSReactiveTableViewController](https://github.com/ElectricPeelSoftware/EPSReactiveTableViewController).
- Table view controllers that don't meet either of those criteria should subclass `UITableViewController`.

##### Collection Views

- Collection view controllers with only one section should subclass [EPSReactiveCollectionViewController](https://github.com/ElectricPeelSoftware/EPSReactiveCollectionViewController).
- Collection view controllers with multiple sections should subclass `UICollectionViewController`.
- View controllers which show a form or a login screen should use [EPSCollectionViewFormLayout](https://github.com/ElectricPeelSoftware/EPSCollectionViewFormLayout) for the layout, if possible.
