# Convering Anonymous to Known Customers
## Motivation
In this exercise, we'll integrate SAP Customer Identity Access Management solution and see how to define a new Matching Rule which will help us merge 2 customers together (the anonymous and the known one).
## Walkthrough
* In the `Web Client Application` we created in the previous exercise:
  * Create a new event: "On Login"
    * Add the following field:
      * ciamId: `string`
    * Map it to the field with the same name under the `Profile` schema
  * In the front-end code, extend the `ReportService` to be able to send this event as well
* Define a `PrimaryEmail Matching Rule`:
  * https://help.sap.com/viewer/8438f051ded544d2ba1303e67fc5ff86/PROD/en-US/173429fdcb0f4404b4f6bb4cf293051a.html
* Create a new SAP Customer Data Cloud Application:
  * https://help.sap.com/viewer/8438f051ded544d2ba1303e67fc5ff86/PROD/en-US/05cd8e1ee3dd4adb98a338f734880dd0.html
  * With the following configuration:
    * userKey: AMPGSznzSCGg
    * secret: FIxkGx2qCzxEMCQy6wqVQ30uBKrQPJXf
    * apiKey: 3_qW1SAHjsrc_SM4QN7YIdaoQZcniGL_OZ_XZmHMkFo4ipSobO4hoADyPrOsqw59F7
    * api base url: https://accounts.us1.gigya.com
* Edit the "Get full accounts in batch" event:
  * Map the following fields:
    * firstName -> firstName
    * lastName -> lastName 
    * email -> primaryEmail
    * UID -> ciamId
  * Schedule the polling to run every minute
* Integrate SAP Customer Data Cloud to the front-end code:
  * Add a script for loading its SDK in `index.html`:
```html
<script
      type="text/javascript"
      src="https://cdns.gigya.com/js/gigya.js?apikey=3_qW1SAHjsrc_SM4QN7YIdaoQZcniGL_OZ_XZmHMkFo4ipSobO4hoADyPrOsqw59F7"
    ></script>
```
  * Add an Angular Provider for injecting the SDK to requesting components [via `InjectionToken`].
  * In `AuthService`:
```typescript
constructor(
    private router: Router,
    reportService: ReportService,
    private zone: NgZone,
    @Inject(GIGYA_CIAM) private gigya: any
  ) {
    this.gigya.accounts.addEventHandlers({
      onLogin: (e) => {
        return this.refresh();
      },
      onLogout: (e) => this.subject.next(ANONYMOUS_USER),
    });
    combineLatest([this.user$, this.isLoggedIn$])
      .pipe(
        filter(([_, loggedIn]) => loggedIn),
        map(([user]) => user)
      )
      .subscribe((user) => reportService.onLogin(user));

    this.refresh();
  }

  async refresh() {
    return await this.zone.runOutsideAngular(() => {
      return new Promise((r) =>
        this.gigya.accounts.getAccountInfo({
          callback: (res) => {
            this.subject.next(
              res.errorCode != 0
                ? ANONYMOUS_USER
                : {
                    $key: res.UID,
                    emailId: res.profile?.email,
                    userName: `${res.profile?.firstName} ${res.profile?.lastName}`,
                    firstName: res.profile?.firstName,
                    lastName: res.profile?.lastName,
                    zip: res.profile?.zip,
                    phoneNumber: res.profile?.phone,
                    isAdmin: false,
                  }
            );

            r(res);
          },
        })
      );
    });
  }

  logout() {
    this.gigya.accounts.logout();
  }

  showRegistrationLogin() {
    this.gigya.accounts.showScreenSet({
      screenSet: "Default-RegistrationLogin",
      startScreen: "gigya-login-screen",
    });
  }
```

* You can now register and login in the website