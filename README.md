# SAML Authentication with Keycloak and LDAP for Rails using Devise and OmniAuth

This guide describes how to configure a Rails application to use SAML for authentication via Keycloak, with group membership obtained from LDAP. Group data is included in the SAML response so that Rails apps do not require direct LDAP access.

---

## 1. Run Keycloak in Docker

```bash
docker run -d --name keycloak \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  -p 8080:8080 \
  quay.io/keycloak/keycloak:latest start-dev
```

Access the Keycloak admin UI at:  
http://localhost:8080

Credentials:
- Username: `admin`
- Password: `admin`

---

## 2. Configure LDAP in Keycloak

1. Log in to the Keycloak admin console.
2. Go to **User Federation** > **Add Provider** > **LDAP**.
3. Enter connection details:
   - Connection URL: `ldap://your-ldap-host`
   - Bind DN: e.g., `cn=readonly,dc=example,dc=com`
   - Bind Credential: your LDAP password
   - Users DN: e.g., `ou=Users,dc=example,dc=com`
4. Click **Test Connection** and **Test Authentication**.
5. Save the configuration.
6. Click **Synchronize all users**.

---

## 3. Map LDAP Groups to Keycloak Groups

1. In your LDAP provider settings in Keycloak, go to the **Mappers** tab.
2. Click **Create**.
3. Configure:
   - Mapper Type: `Group-LDAP Mapper`
   - Group DN: `ou=Groups,dc=example,dc=com`
   - Group Name LDAP Attribute: `cn`
   - Membership LDAP Attribute: `member`
   - User Roles Retrieve Strategy: `LOAD ROLES BY MEMBER ATTRIBUTE`
4. Save and click **Synchronize groups**.

---

## 4. Create a SAML Client in Keycloak

1. Go to **Clients** > **Create**.
2. Set:
   - Client ID: `rails-app`
   - Client Protocol: `saml`
   - Root URL: `http://localhost:3000`
3. Click **Save**.
4. Set:
   - Valid Redirect URIs: `http://localhost:3000/users/auth/saml/callback`
   - Assertion Consumer Service POST Binding URL: same as above

---

## 5. Add Group Data to the SAML Assertion

1. In the **rails-app** client, go to the **Mappers** tab.
2. Click **Create**.
3. Configure:
   - Name: `groups`
   - Mapper Type: `Group List`
   - SAML Attribute Name: `groups`
   - Friendly Name: `groups`
   - Single Attribute: `OFF`
4. Click **Save**.

---

## 6. Configure Rails with Devise and OmniAuth-SAML

### Add required gems

In your `Gemfile`:

```ruby
gem 'devise'
gem 'omniauth-saml'
```

Then:

```bash
bundle install
```

---

### Configure Devise

In `config/initializers/devise.rb`:

```ruby
config.omniauth :saml,
  assertion_consumer_service_url: "http://localhost:3000/users/auth/saml/callback",
  issuer: "rails-app",
  idp_sso_target_url: "http://localhost:8080/realms/YOUR_REALM/protocol/saml",
  idp_cert_fingerprint: "XX:XX:XX:...", # Replace with your actual fingerprint
  name_identifier_format: "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress",
  attribute_statements: {
    email: ['email'],
    groups: ['groups']
  }
```

---

### Set up routes

In `config/routes.rb`:

```ruby
devise_for :users, controllers: { omniauth_callbacks: 'users/omniauth_callbacks' }
```

---

### Implement the callback controller

Create `app/controllers/users/omniauth_callbacks_controller.rb`:

```ruby
class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
  def saml
    auth = request.env["omniauth.auth"]
    @user = User.find_or_initialize_by(email: auth.info.email)
    @user.groups = auth.extra[:raw_info].attributes["groups"]
    @user.save!
    sign_in_and_redirect @user
  end
end
```

Ensure your `User` model has a `groups` column (e.g., `t.json :groups`).

---

## 7. Retrieve Keycloak's SAML Certificate Fingerprint

1. Visit the metadata URL:

```
http://localhost:8080/realms/YOUR_REALM/protocol/saml/descriptor
```

2. Copy the X.509 certificate.

3. Generate the fingerprint:

```bash
openssl x509 -in idp_cert.pem -noout -fingerprint -sha1
```

4. Paste the fingerprint into your Devise config.

---

## 8. Outcome

- Rails users authenticate via SAML.
- Keycloak handles user and group lookups via LDAP.
- Group membership is passed in the SAML assertion to your app.
- No direct LDAP access is required by your Rails application.

---

## References

- Devise: https://github.com/heartcombo/devise  
- OmniAuth-SAML: https://github.com/omniauth/omniauth-saml  
- Keycloak: https://www.keycloak.org  
- Keycloak Docker: https://hub.docker.com/r/quay.io/keycloak/keycloak  
