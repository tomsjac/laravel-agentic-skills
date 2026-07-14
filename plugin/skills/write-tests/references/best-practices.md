# Best practices - Functional PHP/Laravel tests

## The AAA pattern in detail

### Arrange — Prepare

- Create only the data needed for the test, nothing more
- Use Laravel factories with explicit attributes for the values under test
- Set up authentication with `actingAs()` if needed
- Prepare the input data (payload, query params)

```php
// Bon — explicite sur ce qui compte
$user = User::factory()->create(['role' => 'editor']);
$article = Article::factory()->for($user)->create(['status' => 'draft']);

// Mauvais — données implicites, on ne sait pas ce qui est testé
$user = User::factory()->create();
$article = Article::factory()->create();
```

### Act — Act

- A SINGLE action per test (one HTTP call, one method call)
- Capture the result in a clearly named variable (`$response`, `$result`)
- Never mix Act and Assert on the same line

```php
// Bon — action claire et isolée
$response = $this->actingAs($user)->post('/api/articles', $payload);

// Mauvais — deux actions dans le même test
$this->actingAs($user)->post('/api/articles', $payload);
$response = $this->actingAs($user)->get('/api/articles');
```

### Assert — Verify

- Check the HTTP status code AND the database state
- Use specific assertions:
  - `assertSame` > `assertEquals` (strict comparison)
  - `assertCount` > `assertTrue(count(...) === N)`
  - `assertDatabaseHas` / `assertDatabaseMissing` for the database
  - `assertJsonStructure` / `assertJsonFragment` for JSON responses
- No more than 5 assertions per test (beyond that, split the test)

## Factories

### Correct usage

```php
// Créer avec des attributs spécifiques au test
$user = User::factory()->create(['email' => 'test@example.com']);

// Relations
$post = Post::factory()->for($user)->create();

// Multiples enregistrements
$comments = Comment::factory()->count(3)->for($post)->create();

// États prédéfinis
$user = User::factory()->admin()->create();
$post = Post::factory()->published()->create();
```

### Creating missing factories

If a factory does not exist, create it in `database/factories/` following these rules:
- Realistic default values via `fake()`
- States (`state()`) for common variants
- Relations via `for()` / `has()`

## Common Laravel assertions

### HTTP response

```php
$response->assertOk();                    // 200
$response->assertCreated();               // 201
$response->assertNoContent();             // 204
$response->assertNotFound();              // 404
$response->assertForbidden();             // 403
$response->assertUnauthorized();          // 401
$response->assertUnprocessable();         // 422
$response->assertRedirect('/login');      // 302
```

### JSON

```php
$response->assertJson(['status' => 'success']);
$response->assertJsonFragment(['name' => 'John']);
$response->assertJsonStructure(['data' => ['id', 'name', 'email']]);
$response->assertJsonCount(3, 'data');
$response->assertJsonPath('data.0.name', 'John');
$response->assertJsonMissing(['error']);
```

### Database

```php
$this->assertDatabaseHas('users', ['email' => 'test@example.com']);
$this->assertDatabaseMissing('users', ['email' => 'deleted@example.com']);
$this->assertDatabaseCount('users', 5);
$this->assertSoftDeleted('posts', ['id' => $post->id]);
$this->assertModelExists($user);
$this->assertModelMissing($deletedUser);
```

### Events, Jobs, Notifications

```php
Event::fake();
// ... action ...
Event::assertDispatched(OrderShipped::class);
Event::assertNotDispatched(OrderCancelled::class);

Notification::fake();
// ... action ...
Notification::assertSentTo($user, InvoicePaid::class);

Queue::fake();
// ... action ...
Queue::assertPushed(ProcessPodcast::class);
```

## Mocks — When and how

### When to mock (legitimate cases)

- **External APIs**: third-party HTTP services (Stripe, SendGrid, etc.)
- **Email sending**: `Mail::fake()`
- **Queues**: `Queue::fake()` to verify that a job is dispatched
- **Notifications**: `Notification::fake()`
- **Events**: `Event::fake()` only when you want to verify the dispatch without triggering the listeners
- **Time**: `$this->travel()` or `Carbon::setTestNow()` for date-related tests

### When NOT to mock

- **Repositories / internal Services** — use the real implementation with the database
- **Eloquent / Query Builder** — use `RefreshDatabase` with a real database
- **Validation** — test through the full HTTP request
- **Middleware** — test through the full HTTP request
- **Policies** — test through the HTTP request to verify the real behavior

### Example of a legitimate mock

```php
public function test_order_creation_calls_payment_gateway(): void
{
    // Arrange
    $user = User::factory()->create();
    $product = Product::factory()->create(['price' => 2999]);

    Http::fake([
        'api.stripe.com/*' => Http::response(['id' => 'ch_123', 'status' => 'succeeded'], 200),
    ]);

    // Act
    $response = $this->actingAs($user)->post('/api/orders', [
        'product_id' => $product->id,
    ]);

    // Assert
    $response->assertCreated();
    Http::assertSent(fn ($request) => $request->url() === 'https://api.stripe.com/v1/charges');
    $this->assertDatabaseHas('orders', [
        'user_id' => $user->id,
        'status' => 'paid',
    ]);
}
```

## Test naming

### Convention

```
test_<sujet>_<action_ou_condition>_<résultat_attendu>
```

### Examples

```php
// Bon — décrit le comportement complet
test_admin_can_delete_published_article()
test_guest_cannot_access_dashboard()
test_user_registration_with_invalid_email_returns_validation_error()
test_order_total_includes_shipping_when_above_threshold()

// Mauvais — trop vague
test_delete()
test_validation()
test_it_works()
test_user_test()
```

## Test file structure

### Recommended organization

```
tests/
├── Feature/
│   ├── Auth/
│   │   ├── LoginTest.php
│   │   └── RegistrationTest.php
│   ├── Api/
│   │   ├── PostControllerTest.php
│   │   └── CommentControllerTest.php
│   └── Admin/
│       └── UserManagementTest.php
└── Unit/
    └── Services/
        └── PriceCalculatorTest.php
```

### One test file per tested class/controller

Group tests by feature, not by assertion type.

## Data providers

Use data providers to test multiple variants without duplicating code:

```php
public static function invalidEmailProvider(): array
{
    return [
        'missing @' => ['invalidemail.com'],
        'missing domain' => ['user@'],
        'empty string' => [''],
        'spaces only' => ['   '],
    ];
}

#[DataProvider('invalidEmailProvider')]
public function test_registration_rejects_invalid_email(string $email): void
{
    // Arrange
    $payload = User::factory()->make(['email' => $email])->toArray();
    $payload['password'] = 'SecurePass123!';
    $payload['password_confirmation'] = 'SecurePass123!';

    // Act
    $response = $this->post('/api/register', $payload);

    // Assert
    $response->assertUnprocessable();
    $response->assertJsonValidationErrors('email');
}
```
</content>
