# Tumblr SDK for iOS

An unopinionated and flexible library for easily integrating Tumblr data into your iOS or OS X application, however you see fit. The library uses ARC and requires iOS 5 or OS X 10.7.

    [[TMAPIClient sharedInstance] blogInfo:@"bryan" success:^ (id result, NSError *error) {
        if (error) {
          NSLog(@"Bummer, dude: %@", error);
          return;
        }
        NSLog(@"Blog description: %@", result[@"description"]);
    }];

The primary features of the SDK currently include:

* [Authentication](#authentication) (both three-legged OAuth and xAuth)
* [A full API client](#api-client)
* [Inter-app communication](#inter-app-communication) (if the user has the Tumblr iOS app installed)
* [A UIActivity stub](https://github.com/tumblr/tumblr-ios-sdk/blob/master/TMTumblrSDK/TMTumblrActivity.h) (for displaying a Tumblr button in a UIActivityViewController)

Additional questions can be answered on our [discussion group](https://groups.google.com/group/tumblr-api/). Please use the Tumblr API [responsibly](http://www.tumblr.com/docs/en/api_agreement) and [let us know](mailto:api@tumblr.com) what you think.

If you have any feature requests, please let us know by creating an issue or submitting a pull request.

## Getting started

[CocoaPods](http://cocoapods.org) is the recommended way to add the Tumblr SDK to your project. You can simply create a [podfile](https://github.com/CocoaPods/CocoaPods/wiki/A-Podfile) that looks as follows:

    platform :ios, '5.0'

    pod 'TumblrSDK', '1.0.0'

The SDK includes a UIActivity subclass for including Tumblr in a standard UIActivityViewController. It currently provides only the activity icon and title, but you can hook it up however you see fit and we may provide a more integrated solution in the future.

## Authentication

Import `TMAPIClient.h`. Configure the `[TMAPIClient sharedInstance]` singleton with your app’s Tumblr consumer key and secret:

    [TMAPIClient sharedInstance].OAuthConsumerKey = @"ADISJdadsoj2dj38dj29dj38jd9238jdk92djasdjASDaoijsd";
    [TMAPIClient sharedInstance].OAuthConsumerSecret = @"MGI39kdasdoka3240989ASFjoiajsfomdasd39129ASDAPDOJa";

If you don't already have a consumer key/secret you can register [here](http://www.tumblr.com/oauth/apps).

### OAuth
In your app’s `Info.plist`, specify a custom URL scheme that the browser can use to return to your application once the user has permitted or denied access to Tumblr:

    <key>CFBundleURLTypes</key>
    <array>
      <dict>
        <key>CFBundleURLSchemes</key>
        <array>
          <string>myapp</string>
        </array>
      </dict>
    </array>

In your app delegate, allow the `TMAPIClient` singleton to handle incoming URL requests. On iOS this looks like: 

    - (BOOL)application:(UIApplication *)application openURL:(NSURL *)url sourceApplication:(NSString *)sourceApplication
             annotation:(id)annotation {
        return [[TMAPIClient sharedInstance] handleOpenURL:url];
    }
    
And on OS X:
    
    - (void)applicationWillFinishLaunching:(NSNotification *)notification {
        NSAppleEventManager *appleEventManager = [NSAppleEventManager sharedAppleEventManager];
        [appleEventManager setEventHandler:self andSelector:@selector(handleURLEvent:withReplyEvent:)
                             forEventClass:kInternetEventClass andEventID:kAEGetURL];
    }

    - (void)handleURLEvent:(NSAppleEventDescriptor*)event withReplyEvent:(NSAppleEventDescriptor*)replyEvent {
        NSString *calledURL = [[event paramDescriptorForKeyword:keyDirectObject] stringValue];

        [[TMAPIClient sharedInstance] handleOpenURL:[NSURL URLWithString:calledURL]];
    }

Initiate the three-legged OAuth flow, by specifying the URL scheme that your app will respond to:

    [[TMAPIClient sharedInstance] authenticate:@"myapp" callback:^(NSError *error) {
        // You are now authenticated (if !error)
    }];

### xAuth
Please note that xAuth access [must be specifically requested](http://www.tumblr.com/oauth/apps) for your application.

Use the `TMAPIClient` singleton to retrieve an OAuth token and secret given a user’s email address and password:

    [[TMAPIClient sharedInstance] xAuth:@"foo@foo.bar" password:@"12345" callback:^(NSError *error) {
        // You are now authenticated (if !error)
    }];

If you're only interested in authentication, the `TumblrSDK/Authentication` sub-pod can be installed by itself.

## API client

Please view the [API documentation]((http://www.tumblr.com/docs/en/api/v2) for full usage instructions.

There are two ways of retrieving data from the API:

    // `void` methods for immediate requests, preferable when the caller does not need a reference to an actual request object:

    [[TMAPIClient sharedInstance] userInfo:^(id result, NSError *error) {
        if (!error) 
            NSLog(@"Got some user info");
    }];

    // Methods that return configured, signed `JXHTTPOperation` instances and require the client to explicitly send the request separately.

    JXHTTPOperation *likesRequest = [[TMAPIClient sharedInstance] likesRequest:@"bryan" parameters:nil];

    // TODO: Observe some properties, store the request in an instance variable so it can be cancelled if the view controller is deallocated, etc.

    [[TMAPIClient sharedInstance] sendRequest:likesRequest callback:^(id result, NSError *error) {
        if (!error) 
            NSLog(@"Got some liked posts");
    }];

The API client is built on top of the [JXHTTP](https://github.com/jstn/JXHTTP) networking library. If you're only interested in the API client, the `TumblrSDK/APIClient` sub-pod can be installed by itself.

## Inter-app communication

[Tumblr for iOS](https://itunes.apple.com/us/app/tumblr/id305343404?mt=8) exposes actions using the [x-callback-url](http://x-callback-url.com/) specification. The app only supports a few basic endpoints right now but will be added to in the near future:

    TMTumblrAppClient *client = [TMTumblrAppClient client];
    
    if (![client isAppInstalled])
        [client viewInAppStore];

    [client viewDashboard];

    [client viewTag:@"gif"];

    [client viewBlog:@"bryan"];

    [client viewPost:@"43724939726" blogName:@"bryan"];

### URLs

If you don't want to use this library and would rather hit the app's URLs directly, here they are:

    tumblr://x-callback-url/dashboard
    tumblr://x-callback-url/tag?tag=gif
    tumblr://x-callback-url/blog?blogName=bryan
    tumblr://x-callback-url/blog?blogName=bryan&postID=43724939726

Additionally, photos and videos can be passed to Tumblr for iOS using Apple's standard [UIDocumentInteractionController](http://developer.apple.com/library/ios/#documentation/UIKit/Reference/UIDocumentInteractionController_class/Reference/Reference.html).

If you're only interested in the app client, the `TumblrSDK/AppClient` sub-pod can be installed by itself.

### Example

The repository includes a [sample application](https://github.com/tumblr/tumblr-ios-sdk/tree/master/Examples/AppClientExample) showing all of the inter-app hooks in action, as well as how to share to Tumblr for iOS using a UIActivityViewController or UIDocumentInteractionController.

![Screenshot of Tumblr activity icon](https://raw.github.com/tumblr/tumblr-ios-app-client/master/screenshots/uiactivity.png?login=irace&token=ce8dcbc58abd03f1a6006a7fd7abf35e)

## Contact

[Bryan Irace](http://github.com/irace)

## License

Copyright 2012 Tumblr, Inc.

Licensed under the Apache License, Version 2.0 (the “License”); you may not use this file except in compliance with the License. You may obtain a copy of the License at [apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0).

> Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an “AS IS” BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
