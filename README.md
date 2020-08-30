# Security Headers / Secure Cookies / SRI for Production Rails

## Security Headers
### Resources for verifying your headers are correct
[securityheaders.com](https://securityheaders.com/)
[observatory.mozilla.org](https://observatory.mozilla.org//)
It's necessary to uncheck "follow redirects" for sites that use SSO.

### CSP
Read about what CSP is and why it matters check [this out](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP).
CSP is configured for rails in ```config/initalizers/content_security_policy.rb```. Here's an example security policy.

```ruby
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, "code.jquery.com", "cdn.datatables.net" "cdnjs.cloudflare.com"
  policy.style_src   :self, "cdnjs.cloudflare.com" "cdn.datatables.net" "use.fontawesome.com"
  policy.frame_src "*.duosecurity.com 'self'"
end
```
The default policy is require https and only load content from ourselves. So if someone injects ```<script src='malicious.co.uk'></script>``` into the DOM it will violate the CSP and the browser will refuse to load it. Now we may want to include our own js, in this case jquery. So we can add any domain we want to the script_src policy. Any time you load a script or a style from a new domain you'll need to add it to the CSP. Now it's still possible for the script we load at code.jquery.com to change without our knowledge. To prevent this checkout the SRI section.

Many sites at my current job use duo security for MFA. If you don't iframe any other site frame_src none is prefered.

It's also good practice to download the files available at each of the CDNs listed and serve them yourself (and then remove them from the CSP). This prevents your code breaking when cdn.datatables.net goes down. And if you're not using SRI it prevents malicious code from being injected into the page.

### Other Headers
Here's a good default to put in ```config/enviorments/production.rb```
```ruby
  config.action_dispatch.default_headers = {
      'Referrer-Policy' => 'same-origin',
      'X-Content-Type-Options' => 'nosniff',
      'X-Frame-Options' => 'SAMEORIGIN',
      'X-XSS-Protection' => '1; mode=block',
      'Feature-Policy' => "accelerometer 'none'; ambient-light-sensor 'none'; autoplay 'none'; camera 'none'; encrypted-media 'none'; fullscreen 'self'; geolocation 'none'; gyroscope 'none'; magnetometer 'none'; microphone 'none'; midi 'none'; payment 'none'; picture-in-picture 'none'; speaker 'self'; sync-xhr 'none'; usb 'none'; vr 'none'"
  }
```
**Referer-Policy**: "same-origin" means do not tell any OTHER website we link to where the user came from. It is necessary to use same-origin instead of no-referrer as no-referrer breaks links like: ```link_to user_path(@user), method: :delete``` that use a special http action. 

**X-Content-Type-Options:** "nosniff" means that the browser should trust the Content-Type send by the webserver eg. If they download a file called ```user_upload.js.json``` which was supposed to be json and the server believes is json but really contains malicious js form a user the browser will under no circumstances execute the file as javascript and treat it as json as the server directed.

**X-Frame-Options:** "SAMEORIGIN": No one can ```<iframe>``` us but ourselves.

**X-XSS-Protection:** "1; mode=block" Instruct the browser to make its best attempt to prevent XSS and if detected simply stop rendering. 

**Feature-Policy:** '...' Feature policy is a new header governing how the page and children (like other <iframe'd> pages) can access the devices features. Features include things like audio/video/motion/vr. If you need these features you chance this policy but turning them all off is a good default since most sites do not use them. As of September 2019 [there is no way to disable all features without listing them](https://github.com/w3c/webappsec-feature-policy/issues/189).




## Secure Cookie Session Store
Delete ```config/initalizers/session_store.rb``` 
We're going to configure cookies differently for each enviornment in ```config/enviornments/*``` 
For production we prefix the key with *\__Secure* which means it can only be read/written over https.
We also add same_site: strict so that our app's session cookie (differnet from the SSO cookie) isn't sent to anyone else. 
#### Development 
```ruby
Rails.application.configure do
  Rails.application.config.session_store :cookie_store
  ...
end
```
#### Production / Staging
```ruby
Rails.application.configure do
  Rails.application.config.session_store :cookie_store, key: '__Secure-session', same_site: :strict
  ...
end
```

## Script Resource Integrity
To prevent scripts from being changed (eg jquery getting embedded with a key logger at a later date). We can serve a hash of our scripts and stylesheets along with their url. 
So this 
```html
<script src="https://code.jquery.com/jquery-3.4.1.min.js">
```
Becomes
```html
<script src="https://code.jquery.com/jquery-3.4.1.min.js" integrity="sha384-vk5WoKIaW/vJyUAd9n/wmopsmNhiy+L2Z+SBxGYnUkunIxVxAv/UtMOhba/xskxh" crossorigin="anonymous"></script>
```
Then the browser downloads jquery-3.4.1 and if it's hash doesn't match the one we provided in the integrity attribute it won't be used. **This may break the page** but that's preferable to running unknown code. A great resource to generate these easily is [www.srihash.org/](https://www.srihash.org)
