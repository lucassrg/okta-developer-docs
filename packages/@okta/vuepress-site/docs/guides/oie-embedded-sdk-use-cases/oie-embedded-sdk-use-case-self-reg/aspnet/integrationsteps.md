### Step 1: Create a sign up link for new users (User clicks sign up link)

Initiation of the self-registration flow begins when the user clicks the
Sign Up link. On the sign-in page, create a **Sign up** link that links to the
create account page you create in the next step.


<div class="common-image-format">

![Sign up](/img/oie-embedded-sdk/oie-embedded-sdk-use-case-simple-self-serv-screen-sign-up.png
 "Sign up")

</div>

> **Note:** The **Sign up** link appears in the following example under the **Continue** button.

### Step 2: Create the create account page (User enters sign-up info)

The next step is for the user to enter basic information (for example, email,
first, and last name). Create a page that accepts this information.

Example:

<div class="common-image-format">

![Create user](/img/oie-embedded-sdk/oie-embedded-sdk-use-case-simple-self-serv-screen-create.png
 "Create user")

</div>

### Step 3: Call RegisterAsync to register the new user (User clicks Register)

When the user clicks **Register**, create a `UserProfile` object and
set its properties with the user profile information captured from the Create account
page. Pass this object into the `IdxClient RegisterAsync` method.

```csharp
var idxAuthClient = new IdxClient();

var userProfile = new UserProfile();
userProfile.SetProperty("firstName", model.FirstName);
userProfile.SetProperty("lastName", model.LastName);
userProfile.SetProperty("email", model.Email);

var registerResponse = await idxAuthClient.RegisterAsync(userProfile);
```

### Step 4: Handle the response from RegisterAsync

If the org's application is properly configured with multiple factors,
`RegisterAsync` should return a response with `AuthenticationStatus` equal
to `AwaitingAuthenticatorEnrollment`. This status indicates that there is a
required authenticator that needs to be verified. If you completed the steps
properly in the

[Set up your Okta org (for multifactor use cases)](/docs/guides/oie-embedded-common-org-setup/aspnet/main/#set-up-your-okta-org-for-multi-factor-use-cases) section, the authenticator is the **password** factor that
is stored in the `Authenticators` list property.

```csharp
if (registerResponse.AuthenticationStatus == AuthenticationStatus.AwaitingAuthenticatorEnrollment)
    {
       Session["idxContext"] = registerResponse.IdxContext;
       TempData["authenticators"] = ViewModelHelper.
             ConvertToAuthenticatorViewModelList(registerResponse.Authenticators);
       return RedirectToAction("SelectAuthenticator", "Manage");
    }
```

### Step 5: Build an authenticator list page (Show password factor to user)

The next step is to build a page that shows the required factors that need to
be verified. After the call to `RegisterAsync`, the user needs to see the
password factor requirement and select it for verification.

<div class="common-image-format">

![Verify password](/img/oie-embedded-sdk/oie-embedded-sdk-use-case-simple-self-serv-screen-verify-password.png
 "Verify password")

</div>

The page should be generic and handle the list of authenticators that are returned from
the various methods of the SDK. Use the `Authenticators` property in the
returned response to build the list.

```csharp
var authenticators = (IList<AuthenticatorViewModel>)TempData["authenticators"];
var viewModel = new SelectAuthenticatorViewModel
  {
     Authenticators = authenticators,
     PasswordId = authenticators.FirstOrDefault(x => x.Name.ToLower() == "password")?.AuthenticatorId,
     PhoneId = authenticators.FirstOrDefault(x => x.Name.ToLower() == "phone")?.AuthenticatorId,
     CanSkip = TempData["canSkip"] != null && (bool)TempData["canSkip"]
  };

TempData["authenticators"] = viewModel.Authenticators;
return View(viewModel);
```

### Step 6: Call EnrollAuthenticatorAsync (User selects password authenticator)

The next step is to call the `EnrollAuthenticatorAsync` method when the user
selects the authenticator. In this case, the `AuthenticatorId` for the
**password** factor is passed in.

```csharp
var enrollAuthenticatorOptions = new EnrollAuthenticatorOptions
{
     AuthenticatorId = model.AuthenticatorId,
};

var enrollResponse = await idxAuthClient.EnrollAuthenticatorAsync(enrollAuthenticatorOptions, (IIdxContext)Session["IdxContext"]);
```

### Step 7: Handle the response to EnrollAuthenticatorAsync

`EnrollAuthenticatorAsync` returns an `AuthenticationStatus`, and if the
enrollment is successful, this property should return
`AwaitingAuthenticatorVerification`. When `AwaitingAuthenticatorVerification`
is returned, the next step is to verify the authenticator. In this case, the
user needs to verify with the **password** authenticator.

```csharp
switch (enrollResponse?.AuthenticationStatus)
{
   case AuthenticationStatus.AwaitingAuthenticatorVerification:
   if (model.IsPasswordSelected)
   {
      return RedirectToAction("ChangePassword", "Manage");
   }

   return RedirectToAction("VerifyAuthenticator", "Manage");

case AuthenticationStatus.AwaitingAuthenticatorEnrollmentData:
   return RedirectToAction("EnrollPhoneAuthenticator", "Manage");
default:
   return View("SelectAuthenticator", model);
}
```

### Step 8: Build page to verify new password authenticator (Show user password fields)

After the password authenticator is awaiting verification, the next step
is to build a page that allows the user to verify the new password by
supplying the password.

<div class="common-image-format">

![Confirm password](/img/oie-embedded-sdk/oie-embedded-sdk-use-case-simple-self-serv-screen-confirm-password.png
 "Confirm password")

</div>

### Step 9: Call VerifyAuthenticatorAsync to verify password (User enters password)

When the user fills out the new password and clicks **Register**, a call
to `VerifyAuthenticatorAsync` is made to verify (in this case, to set the password for
the new user). Use the `Code` property in the `VerifyAuthenticatorOptions` parameter
to store the new password.

```csharp
var idxAuthClient = new IdxClient(null);
           var verifyAuthenticatorOptions = new VerifyAuthenticatorOptions
           {
               Code = code,
           };

var authnResponse = await idxAuthClient.VerifyAuthenticatorAsync(verifyAuthenticatorOptions, (IIdxContext)Session["idxContext"]);
```

### Step 10: Handle the response from VerifyAuthenticatorAsync (Starting multi-factor verification flow)

If you completed the steps in

[Set up your Okta org (for multifactor use cases)](/docs/guides/oie-embedded-common-org-setup/aspnet/main/#set-up-your-okta-org-for-multi-factor-use-cases)
that sets up multifactors for your application, the return

`AuthenticationResponse.AuthenticationStatus` should be
`AwaitingAuthenticatorEnrollment`. The `AwaitingAuthenticatorEnrollment`
status is returned because the required **email** and optional **phone**
factors await to be enrolled and verified. The user should be redirected to an authenticator list page.

<div class="common-image-format">

![Authenticator list](/img/oie-embedded-sdk/oie-embedded-sdk-use-case-simple-self-serv-screen-auth-list-email-phone.png
 "Authenticator List")

</div>

> **Note:** In the previous screenshot, the **Skip** button is used to skip the
authenticators. You can implement a **Skip** button on your authenticators
list page when one of the authenticators is optional. Since the email
factor is required, but the phone is **optional**, the **Skip** button in this
step is visible, yet disabled. The **Skip** button is described in later steps.

The code snippet below shows how the response is handled.
`AwaitingAuthenticatorEnrollment` identifies that there are additional
factors (in this case, email and optionally phone). The Authenticator list
page is loaded again (first time was for password) with the two additional
factors.

```csharp
switch (authnResponse.AuthenticationStatus)
   {
       ...
       case AuthenticationStatus.AwaitingAuthenticatorEnrollment:
           TempData["authenticators"] =
              ViewModelHelper.ConvertToAuthenticatorViewModelList(authnResponse.Authenticators);
              TempData["canSkip"] = authnResponse.CanSkip;
            return RedirectToAction("SelectAuthenticator", "Manage");
...
```

> **Note** The `CanSkip` property in the code sample above is used for optional factors. See the SDK sample for more information.

### Step 11: Call EnrollAuthenticatorAsync to verify the email authenticator  (User selects email factor)

Assuming that the user selects the **email** authenticator, a call to
`EnrollAuthenticatorAsync` is made passing in the **email**
`AuthenticatorId`. If successful, this call should send a code to the
user's email.

```csharp
var enrollAuthenticatorOptions = new EnrollAuthenticatorOptions
{
     AuthenticatorId = model.AuthenticatorId,
};

var enrollResponse = await idxAuthClient.EnrollAuthenticatorAsync(enrollAuthenticatorOptions, (IIdxContext)Session["IdxContext"]);
```

### Step 12: Update the Email Factor Verification email template (optional)

The email sent to the user has a **Verify Email Address** link that isn't yet
supported. Accordingly, there are two recommended options to mitigate this limitation.
See
[Email verify email link does not work](/docs/guides/oie-embedded-sdk-limitations/main/#email-verify-email-link-does-not-work).

### Step 13: Handle the response to EnrollAuthenticatorAsync

If the call to `EnrollAuthenticatorAsync` was successful, it should return
an `AuthenticationStatus` of `AwaitingAuthenticatorVerification`. When
`AwaitingAuthenticatorVerification` is returned, a code is sent to the user's
email, and the user needs to verify this code.

The following sample app code snippet shows that the user is redirected to the
**Verify Authenticator** page to verify that the code was sent in the email.

```csharp
var enrollResponse = await idxAuthClient.EnrollAuthenticatorAsync(enrollAuthenticatorOptions, (IIdxContext)Session["IdxContext"]);

switch (enrollResponse?.AuthenticationStatus)
{
   case AuthenticationStatus.AwaitingAuthenticatorVerification:
        return RedirectToAction("VerifyAuthenticator", "Manage");
...
}
```

### Step 14: Build out email verification code page (Show user email verification page)

Build out the email verification code page that accepts the code from the email.

<div class="common-image-format">

![Authenticator list](/img/oie-embedded-sdk/oie-embedded-sdk-use-case-simple-self-serv-screen-verify-email-code.png
 "Authenticator List")

</div>

### Step 15: Call VerifyAuthenticatorAsync to verify email code (User enters code and clicks verify)

The next step is to call `VerifyAuthenticatorAsync`. In the case of the
email verification, the code passed into `VerifyAuthenticatorAsync` is the code found in the verification email.

```csharp
var idxAuthClient = new IdxClient(null);
           var verifyAuthenticatorOptions = new VerifyAuthenticatorOptions
           {
               Code = code,
           };

var authnResponse = await idxAuthClient.VerifyAuthenticatorAsync(verifyAuthenticatorOptions, (IIdxContext)Session["idxContext"]);
```

### Step 16: Handle response from VerifyAuthenticatorAsync

The next step is to handle the response from `VerifyAuthenticatorAsync`.
If the email code was valid, the method should return `AuthenticationStatus`
of `AwaitingAuthenticatorEnrollment`. This status signifies that there is
another factor (required or optional) waiting to be enrolled and verified.
If the steps described in
[Set up your Okta org (for multi-factor use cases)](/docs/guides/oie-embedded-common-org-setup/aspnet/main/#set-up-your-okta-org-for-multifactor-use-cases) were properly followed,
the user should be sent back to the Authenticator list page
showing only the **phone** authenticator.

```csharp
var authnResponse = await _idxClient.VerifyAuthenticatorAsync(verifyAuthenticatorOptions,
     (IIdxContext)Session["idxContext"]);
Session["idxContext"] = authnResponse.IdxContext;

switch (authnResponse.AuthenticationStatus)
      {
          case AuthenticationStatus.AwaitingAuthenticatorEnrollment:
               TempData["authenticators"] =
               ViewModelHelper.ConvertToAuthenticatorViewModelList
               (authnResponse.Authenticators);
               TempData["canSkip"] = authnResponse.CanSkip;
               return RedirectToAction("SelectAuthenticator", "Manage");
           ...
      }
```

### Step 17: Show remaining list of authenticators (Show user the phone factor)

The remaining authenticator should display the phone factor. Since this factor
is optional now and there are no other required factors that need to be verified,
the user should have the ability to skip the factor. Create a **Skip** button
for this use case. This **Skip** button is governed by the `CanSkip`
property on the `AuthenticationResponse`. See the following screenshot for an
illustration.

<div class="common-image-format">

![Phone list](/img/oie-embedded-sdk/oie-embedded-sdk-use-case-simple-self-serv-screen-auth-list-phone.png
 "Phone List")

</div>

The user can either enroll in the phone factor or skip the phone factor. Your
code should handle both scenarios described in the following steps.

### Step 18a: Option 1: Enroll and verify the phone authenticator (User selects phone factor)

If the user selects the phone authenticator (instead of skipping it), the
steps to enroll and verify are similar to the email factor verification flow in
this section with subtle differences.

### Step 18b: Option 1: Call EnrollAuthenticatorAsync (1st time) to start phone verification

If the user selects the **phone** authenticator, a call to
`EnrollAuthenticatorAsync` is made passing in the **phone**
`AuthenticatorId`. If successful, the method should return an
`AwaitingAuthenticatorEnrollmentData` response. The `AwaitingAuthenticatorEnrollmentData `response indicates that the enrollment data is required before continuing to verification. In the case of the phone
authenticator, the phone number is required, and the user should be redirected
to a page where they can enter in a phone number. See the following code snippet from the sample app.

```csharp
var enrollResponse = await _idxClient.EnrollAuthenticatorAsync(enrollAuthenticatorOptions,
     (IIdxContext)Session["IdxContext"]);
...

switch (enrollResponse?.AuthenticationStatus)
     {
          ...
          case AuthenticationStatus.AwaitingAuthenticatorEnrollmentData:
               return RedirectToAction("EnrollPhoneAuthenticator", "Manage");
          ...
      }
```

### Step 18c: Option 1: Build out phone number entry page (Show user phone number entry page

Build out the phone number entry page that accepts the phone number that the
user then uses to enroll and verify.

<div class="common-image-format">

![Verify phone](/img/oie-embedded-sdk/oie-embedded-sdk-use-case-simple-self-serv-screen-verify-phone-num.png
 "Verify phone")

</div>

> **Note:** The SDK requires that the phone number format be in the following format: `+#######`, including the beginning plus (+) sign. See
[Data Requirements - Phone number](/docs/guides/oie-embedded-sdk-common/aspnet/main/#phone-number).

### Step 18d: Option1: Call EnrollAuthenticatorAsync (2nd time) to submit phone number and send SMS (User clicks send code via SMS button)

When the user enters their phone number and clicks the send code using the SMS button, a call to `EnrollAuthenticatorAsync` is made passing in the following values:

1. Authenticator ID
1. Phone number
1. Method type (only SMS is currently supported)

> **Note** Only SMS is currently supported for the phone authenticator type.

The above values are passed in using the `EnrollPhoneAuthenticatorOptions` parameter.
See the following code snippet for more details.

```csharp
 var enrollPhoneAuthenticatorOptions = new EnrollPhoneAuthenticatorOptions
     {
        AuthenticatorId = Session["phoneId"].ToString(),
        PhoneNumber = model.PhoneNumber,
        MethodType = AuthenticatorMethodType.Sms,
     };

var enrollResponse = await _idxClient.EnrollAuthenticatorAsync(enrollPhoneAuthenticatorOptions,
      (IIdxContext)Session["IdxContext"]);
      Session["IdxContext"] = enrollResponse.IdxContext;
```

### Step 18e: Option1: Handle the response to EnrollAuthenticatorAsync

If the call to `EnrollAuthenticatorAsync` is successful, it should
return an `AuthenticationStatus` of `AwaitingAuthenticatorVerification`.
When `AwaitingAuthenticatorVerification` is returned, a code is sent to the
phone number through SMS.

In the following code snippet, the user is redirected to a reusable code
verification page that handles the code for both email and SMS. Your
implementation may vary.

```csharp
var enrollResponse = await _idxClient.EnrollAuthenticatorAsync(enrollPhoneAuthenticatorOptions,
    (IIdxContext)Session["IdxContext"]);
    ...
if (enrollResponse.AuthenticationStatus ==
    AuthenticationStatus.AwaitingAuthenticatorVerification)
    {
       return RedirectToAction("VerifyAuthenticator", "Manage");
    }
```

### Step 18f: Option1: Build out (or reuse) phone verification code page (Show user phone verification page)

Build out a page that accepts the code sent to your phone number through SMS.
Depending on your implementation, this page can be the same page that
verifies the email code or different. The sample app reuses the same page
for both email and phone verification.

<div class="common-image-format">

![Enter code from phone](/img/oie-embedded-sdk/oie-embedded-sdk-use-case-simple-self-serv-screen-verify-phone-code.png
 "Enter code from phone")

</div>

### Step 18g: Option 1: Call VerifyAuthenticatorAsync to verify phone code (User enters code and clicks verify)

The next step is to call `VerifyAuthenticatorAsync`. In the case of the
phone verification, the code passed into `VerifyAuthenticatorAsync` is the code that was sent through SMS to the phone number.

```csharp
var idxAuthClient = new IdxClient(null);
           var verifyAuthenticatorOptions = new VerifyAuthenticatorOptions
           {
               Code = code,
           };

var authnResponse = await idxAuthClient.VerifyAuthenticatorAsync(verifyAuthenticatorOptions, (IIdxContext)Session["idxContext"]);
```

### Step 18h: Option 1: Handle response from VerifyAuthenticatorAsync (Factor verifications completed)

The next step is to handle the response from `VerifyAuthenticatorAsync`. If
the phone SMS code was valid, the method should return `AuthenticationStatus`
of `Success`. This status signifies that there are no more factors (required or
optional) waiting to be enrolled and verified. If the steps described in

[Set up your Okta org (for multifactor use cases)](/docs/guides/oie-embedded-common-org-setup/aspnet/main/#set-up-your-okta-org-for-multi-factor-use-cases)
were properly followed, the user should now be registered with no more
factors to be verified. The user should then be sent to the default page after they have successfully registered. In the case of the sample application, the user
is sent to the user profile page.

```csharp
var authnResponse = await _idxClient.VerifyAuthenticatorAsync(verifyAuthenticatorOptions, (IIdxContext)Session["idxContext"]);
Session["idxContext"] = authnResponse.IdxContext;

switch (authnResponse.AuthenticationStatus)
{
   ...
   case AuthenticationStatus.Success:
         ClaimsIdentity identity = await
         AuthenticationHelper.GetIdentityFromTokenResponseAsync(_idxClient.Configuration,
         authnResponse.TokenInfo);
         _authenticationManager.SignIn(new AuthenticationProperties(), identity);
         return RedirectToAction("Index", "Home");
   ...
}
```

### Step 19: Option 2: Call SkipAuthenticatorSelectionAsync (User clicks skip)

If the user opts to skip the phone enrollment, a call to
`SkipAuthenticatorSelectionAsync` needs to be made. This method skips the
phone enrollment and eliminates the need to verify the factor. See the following code
snippet for more details.

```csharp
var skipSelectionResponse = await _idxClient.SkipAuthenticatorSelectionAsync((IIdxContext)Session["IdxContext"]);
switch (skipSelectionResponse.AuthenticationStatus)
    {
         case AuthenticationStatus.Success:
              ClaimsIdentity identity = await
              AuthenticationHelper.GetIdentityFromTokenResponseAsync(_idxClient.Configuration,
              skipSelectionResponse.TokenInfo);
              _authenticationManager.SignIn(new AuthenticationProperties(), identity);
              return RedirectToAction("Index", "Home");
         case AuthenticationStatus.Terminal:
              TempData["MessageToUser"] = skipSelectionResponse.MessageToUser;
              return RedirectToAction("Login", "Account");
   }
```

The method `SkipAuthenticatorSelectionAsync` can return different response
statuses: `Success` or `Terminal`.  For `Success`, the user is signed in, the response
is stored in session, and the user is redirected to the default sign
in page (in the case of the sample app, it's the user profile page).

### Step 20: User is sent to user profile page

After the factor verifications are successful, and there are no more
authenticators to enroll and verify, the user is now successfully
registered and can be sent to the default sign in page. In the case
of the sample app, the default sign page is the user profile page.
See
[Get user profile information after sign in](/docs/guides/oie-embedded-sdk-alternate-flows/aspnet/main/#getuserprofileinfo) for more details on how to fetch user information.

### Troubleshooting Tips

Ensure that when testing this use case, you use a new email each time. If you have
a gmail account, you can reuse the same email by adding a plus (+) and additional
text (for example, `myemail+1@gmail.com`, `myemail+2@gmail.com`, and so on).
Ensure that the password that you use meets the minimum security requirements. For example,
passwords such as `test123` fail.