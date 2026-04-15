---
name: azure-ad-integration
description: Azure Active Directory OAuth2 authentication and Microsoft Graph API integration. Activate when working on service-internal-auth, AAD login flows, Microsoft Graph user/group queries, or OAuth2 state validation.
metadata:
  tags: azure, aad, oauth2, microsoft-graph, authentication, laravel
---

## Overview

`service-internal-auth` is a dedicated Laravel 12 headless authentication service used by internal tools. It handles Azure AD OAuth2 flows, queries Microsoft Graph for user/group data, and exposes a permission system per service.

**Key packages:**
- `league/oauth2-client` — OAuth2 provider abstraction
- `microsoft/microsoft-graph` — Graph API client
- `laravel/sanctum` — API token authentication

---

## OAuth2 Flow

### Step 1 — Sign In (get auth URL)
```php
// GET /signin
public function signIn(Request $request): JsonResponse
{
    $authUrl = $this->provider->getAuthorizationUrl();
    Cache::put('oauth2_state_' . $request->ip(), $this->provider->getState(), 300);

    return response()->json(['url' => $authUrl]);
}
```

### Step 2 — Callback (exchange code for token)
```php
// GET /{service}/user?code=...&state=...
public function user(string $service, Request $request): JsonResponse
{
    // Validate state to prevent CSRF
    $cachedState = Cache::get('oauth2_state_' . $request->ip());

    if ($request->state !== $cachedState) {
        abort(401, 'Invalid OAuth state');
    }

    $token = $this->provider->getAccessToken('authorization_code', [
        'code' => $request->code,
    ]);

    $adUser = $this->getUser($token);
    $adGroups = $this->getGroups($token);

    $user = User::updateOrCreate(
        ['email' => $adUser->getMail()],
        ['name' => $adUser->getDisplayName()]
    );

    return response()->json([
        'access_token' => $token->getToken(),
        'refresh_token' => $token->getRefreshToken(),
        'expires' => $token->getExpires(),
        'user_name' => $adUser->getDisplayName(),
        'user_mail' => $adUser->getMail(),
        'groups' => $adGroups,
    ]);
}
```

### Step 3 — Refresh token
```php
// GET /{service}/refresh?refresh_token=...
public function refresh(string $service, Request $request): JsonResponse
{
    $token = $this->provider->getAccessToken('refresh_token', [
        'refresh_token' => $request->refresh_token,
    ]);

    return response()->json([
        'access_token' => $token->getToken(),
        'expires' => $token->getExpires(),
    ]);
}
```

---

## Microsoft Graph API Queries

### Get authenticated user
```php
private function getUser(AccessToken $token): AdUser
{
    $this->graph->setAccessToken($token->getToken());

    return $this->graph
        ->createRequest('GET', '/me?$select=displayName,mail,userPrincipalName')
        ->setReturnType(AdUser::class)
        ->execute();
}
```

### Get user's AD groups
```php
private function getGroups(AccessToken $token): array
{
    $this->graph->setAccessToken($token->getToken());

    return $this->graph
        ->createRequest('GET', '/me/memberOf?$select=id,displayName&$top=100')
        ->setReturnType(AdGroup::class)
        ->execute();
}
```

Always select only needed fields (`$select`) — Graph responses can be large.
Use `$top=100` for group queries — default page size is 10.

---

## Permission System

Permissions are stored per service per user email:

```php
// GET /{service}/permissions/{email}
public function getUserPermissions(string $service, string $email): JsonResponse
{
    $permissions = Service::getPermissionsByUserEmail($service, $email);
    return response()->json($permissions);
}

// POST /{service}/permissions
public function store(string $service, Request $request): JsonResponse
{
    // Grant permission to a user for a service
}
```

---

## Models

```php
// User — synced from Azure AD on each login
User::updateOrCreate(
    ['email' => $adUser->getMail()],
    ['name' => $adUser->getDisplayName()]
);

// Service — maps service slug to permissions
Service::getPermissionsByUserEmail(string $service, string $email): array

// LogEntry — audit log per service
LogEntry::create(string $service, string $userName, string $action);
```

---

## Routes

```php
Route::get('/signin', [AuthController::class, 'signIn']);
Route::get('/{service}/user', [AuthController::class, 'user']);
Route::get('/{service}/refresh', [AuthController::class, 'refresh']);
Route::post('/{service}/log', [LogController::class, 'store']);
Route::get('/{service}/permissions', [ServiceController::class, 'get']);
Route::post('/{service}/permissions', [ServiceController::class, 'store']);
Route::get('/{service}/permissions/{email}', [AuthController::class, 'getUserPermissions']);
```

---

## Key Rules

- Always validate OAuth state from cache before exchanging code — prevents CSRF
- Cache state keyed by IP (`oauth2_state_{ip}`) with 5-minute TTL
- Use `$select` in all Graph queries — never fetch full objects
- `$top=100` on group queries — users can be in many groups
- Store state in Cache, not session — this is a stateless API service
- `GenericProvider` from `league/oauth2-client` is used (not a Microsoft-specific provider) — Azure AD endpoints are configured via `.env`
