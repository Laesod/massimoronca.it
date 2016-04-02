---
draft: true
---

**Gemfile**

```ruby

gem 'devise_saml_authenticatable'

group :test do
  gem 'saml-server', github: 'consolo/saml-server'
end
```

### Environment variables

```bash
SAML_CONSUMER_SERVICE_URL=/<devise-endpoint>/saml/auth
SAML_ISSUER=/<devise-endpoint>/saml/metadata
SAML_SLO_TARGET_URL=/idp/saml
SAML_SSO_TARGET_URL=/idp/saml
# for brevity, but you can also use the complete cert
SAML_CERT_FINGERPRINT=9E:65:2E:03:06:8D:80:F2:86:C7:6C:77:A1:D9:14:97:0A:4D:F4:4D
```

### Configure Devise

**devise.rb**

```ruby
Devise.setup do |config|
  # regular Devise configuration...

  # ==> Configuration for :saml_authenticatable

   # Create user if the user does not exist. (Default is false)
   config.saml_create_user = true
   # Update the attributes of the user after a successful login. (Default is false)
   config.saml_update_user = true
   # Set the default user key. The user will be looked up by this key. Make
   # sure that the Authentication Response includes the attribute.
   config.saml_default_user_key = :email
   # Optional. This stores the session index defined by the IDP during login.  If provided it will be used as a salt
   # for the user's session to facilitate an IDP initiated logout request.
   config.saml_session_index_key = :session_index
   # You can set this value to use Subject or SAML assertation as info to which email will be compared
   # If you don't set it then email will be extracted from SAML assertation attributes
   config.saml_use_subject = true

   # Configure with your SAML settings (see [ruby-saml][] for more information).
   config.saml_configure do |settings|
     settings.assertion_consumer_service_url     = ENV["SAML_CONSUMER_SERVICE_URL"]
     settings.assertion_consumer_service_binding = "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
     settings.name_identifier_format             = "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"
     settings.issuer                             = ENV["SAML_ISSUER"]
     settings.authn_context                      = ""
     settings.idp_slo_target_url                 = ENV["SAML_SLO_TARGET_URL"]
     settings.idp_sso_target_url                 = ENV["SAML_SSO_TARGET_URL"]
     if ENV['SAML_APP_CERT'].blank?
       settings.idp_cert_fingerprint             = ENV['SAML_CERT_FINGERPRINT']
     else
       # cert is base64 encoded because ENV variables must be one liners
       settings.idp_cert                         = Base64.decode64 ENV['SAML_APP_CERT']
     end
   end
  end
```

### Rack middleware configuration

**config.ru**

```ruby
if Rails.env.test?
  map '/' do
    run Rails.application
  end

  map '/idp' do
    run SamlServer::App
  end
else
  run Rails.application
end
```
