# Concrete examples — Functional Laravel tests

## Full CRUD test — API Controller

```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Api;

use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class PostControllerTest extends TestCase
{
    use RefreshDatabase;

    // --- INDEX ---

    public function test_authenticated_user_can_list_posts(): void
    {
        // Arrange
        $user = User::factory()->create();
        Post::factory()->count(3)->create();

        // Act
        $response = $this->actingAs($user)->getJson('/api/posts');

        // Assert
        $response->assertOk();
        $response->assertJsonCount(3, 'data');
        $response->assertJsonStructure([
            'data' => [['id', 'title', 'content', 'created_at']],
        ]);
    }

    public function test_guest_cannot_list_posts(): void
    {
        // Arrange — rien, pas d'authentification

        // Act
        $response = $this->getJson('/api/posts');

        // Assert
        $response->assertUnauthorized();
    }

    // --- STORE ---

    public function test_user_can_create_post_with_valid_data(): void
    {
        // Arrange
        $user = User::factory()->create();
        $payload = [
            'title' => 'Mon article',
            'content' => 'Le contenu de mon article.',
        ];

        // Act
        $response = $this->actingAs($user)->postJson('/api/posts', $payload);

        // Assert
        $response->assertCreated();
        $response->assertJsonFragment(['title' => 'Mon article']);
        $this->assertDatabaseHas('posts', [
            'user_id' => $user->id,
            'title' => 'Mon article',
        ]);
    }

    public function test_post_creation_fails_without_title(): void
    {
        // Arrange
        $user = User::factory()->create();
        $payload = ['content' => 'Contenu sans titre.'];

        // Act
        $response = $this->actingAs($user)->postJson('/api/posts', $payload);

        // Assert
        $response->assertUnprocessable();
        $response->assertJsonValidationErrors('title');
    }

    // --- UPDATE ---

    public function test_author_can_update_own_post(): void
    {
        // Arrange
        $user = User::factory()->create();
        $post = Post::factory()->for($user)->create(['title' => 'Ancien titre']);

        // Act
        $response = $this->actingAs($user)->putJson("/api/posts/{$post->id}", [
            'title' => 'Nouveau titre',
        ]);

        // Assert
        $response->assertOk();
        $this->assertDatabaseHas('posts', [
            'id' => $post->id,
            'title' => 'Nouveau titre',
        ]);
    }

    public function test_user_cannot_update_another_users_post(): void
    {
        // Arrange
        $author = User::factory()->create();
        $otherUser = User::factory()->create();
        $post = Post::factory()->for($author)->create();

        // Act
        $response = $this->actingAs($otherUser)->putJson("/api/posts/{$post->id}", [
            'title' => 'Tentative de modification',
        ]);

        // Assert
        $response->assertForbidden();
    }

    // --- DESTROY ---

    public function test_admin_can_delete_any_post(): void
    {
        // Arrange
        $admin = User::factory()->admin()->create();
        $post = Post::factory()->create();

        // Act
        $response = $this->actingAs($admin)->deleteJson("/api/posts/{$post->id}");

        // Assert
        $response->assertNoContent();
        $this->assertDatabaseMissing('posts', ['id' => $post->id]);
    }
}
```

## Authentication test

```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Auth;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class LoginTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_can_login_with_valid_credentials(): void
    {
        // Arrange
        $user = User::factory()->create([
            'email' => 'john@example.com',
            'password' => bcrypt('password123'),
        ]);

        // Act
        $response = $this->postJson('/api/login', [
            'email' => 'john@example.com',
            'password' => 'password123',
        ]);

        // Assert
        $response->assertOk();
        $response->assertJsonStructure(['token']);
    }

    public function test_login_fails_with_wrong_password(): void
    {
        // Arrange
        $user = User::factory()->create([
            'email' => 'john@example.com',
            'password' => bcrypt('password123'),
        ]);

        // Act
        $response = $this->postJson('/api/login', [
            'email' => 'john@example.com',
            'password' => 'wrong_password',
        ]);

        // Assert
        $response->assertUnauthorized();
    }

    public function test_login_fails_with_nonexistent_email(): void
    {
        // Arrange — pas d'utilisateur créé

        // Act
        $response = $this->postJson('/api/login', [
            'email' => 'unknown@example.com',
            'password' => 'password123',
        ]);

        // Assert
        $response->assertUnauthorized();
    }
}
```

## Test with a mocked external service

```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Api;

use App\Models\Order;
use App\Models\Product;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Http;
use Tests\TestCase;

final class OrderCheckoutTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_can_checkout_and_payment_is_processed(): void
    {
        // Arrange
        $user = User::factory()->create();
        $product = Product::factory()->create(['price' => 4999]);

        Http::fake([
            'api.stripe.com/v1/charges' => Http::response([
                'id' => 'ch_test_123',
                'status' => 'succeeded',
            ], 200),
        ]);

        // Act
        $response = $this->actingAs($user)->postJson('/api/checkout', [
            'product_id' => $product->id,
        ]);

        // Assert
        $response->assertCreated();
        $this->assertDatabaseHas('orders', [
            'user_id' => $user->id,
            'product_id' => $product->id,
            'amount' => 4999,
            'status' => 'paid',
        ]);
        Http::assertSent(fn ($request) => str_contains($request->url(), 'api.stripe.com'));
    }

    public function test_checkout_fails_gracefully_when_payment_gateway_errors(): void
    {
        // Arrange
        $user = User::factory()->create();
        $product = Product::factory()->create(['price' => 4999]);

        Http::fake([
            'api.stripe.com/*' => Http::response(['error' => 'card_declined'], 402),
        ]);

        // Act
        $response = $this->actingAs($user)->postJson('/api/checkout', [
            'product_id' => $product->id,
        ]);

        // Assert
        $response->assertUnprocessable();
        $response->assertJsonFragment(['message' => 'Payment failed']);
        $this->assertDatabaseMissing('orders', ['user_id' => $user->id]);
    }
}
```

## Test with time manipulation

```php
<?php

declare(strict_types=1);

namespace Tests\Feature;

use App\Models\Subscription;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class SubscriptionExpirationTest extends TestCase
{
    use RefreshDatabase;

    public function test_subscription_is_active_before_expiration(): void
    {
        // Arrange
        $user = User::factory()->create();
        Subscription::factory()->for($user)->create([
            'expires_at' => now()->addMonth(),
        ]);

        // Act
        $response = $this->actingAs($user)->getJson('/api/subscription/status');

        // Assert
        $response->assertOk();
        $response->assertJsonFragment(['active' => true]);
    }

    public function test_subscription_is_expired_after_expiration_date(): void
    {
        // Arrange
        $user = User::factory()->create();
        Subscription::factory()->for($user)->create([
            'expires_at' => now()->addMonth(),
        ]);

        // Act — avancer dans le temps de 2 mois
        $this->travel(2)->months();
        $response = $this->actingAs($user)->getJson('/api/subscription/status');

        // Assert
        $response->assertOk();
        $response->assertJsonFragment(['active' => false]);
    }
}
```

## Test with file upload

```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Api;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Tests\TestCase;

final class AvatarUploadTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_can_upload_avatar(): void
    {
        // Arrange
        Storage::fake('public');
        $user = User::factory()->create();
        $file = UploadedFile::fake()->image('avatar.jpg', 200, 200);

        // Act
        $response = $this->actingAs($user)->postJson('/api/profile/avatar', [
            'avatar' => $file,
        ]);

        // Assert
        $response->assertOk();
        Storage::disk('public')->assertExists("avatars/{$file->hashName()}");
        $this->assertDatabaseHas('users', [
            'id' => $user->id,
            'avatar_path' => "avatars/{$file->hashName()}",
        ]);
    }

    public function test_avatar_upload_rejects_non_image_file(): void
    {
        // Arrange
        Storage::fake('public');
        $user = User::factory()->create();
        $file = UploadedFile::fake()->create('document.pdf', 1024);

        // Act
        $response = $this->actingAs($user)->postJson('/api/profile/avatar', [
            'avatar' => $file,
        ]);

        // Assert
        $response->assertUnprocessable();
        $response->assertJsonValidationErrors('avatar');
        Storage::disk('public')->assertDirectoryEmpty('avatars');
    }
}
```

## Unit test — Pure business logic

Only when `--unit` is requested or for logic with no Laravel dependency:

```php
<?php

declare(strict_types=1);

namespace Tests\Unit\Services;

use App\Services\PriceCalculator;
use PHPUnit\Framework\TestCase;

final class PriceCalculatorTest extends TestCase
{
    public function test_total_with_tax_applies_correct_rate(): void
    {
        // Arrange
        $calculator = new PriceCalculator();

        // Act
        $result = $calculator->totalWithTax(amount: 10000, taxRate: 20.0);

        // Assert
        $this->assertSame(12000, $result);
    }

    public function test_discount_is_applied_before_tax(): void
    {
        // Arrange
        $calculator = new PriceCalculator();

        // Act
        $result = $calculator->totalWithTax(amount: 10000, taxRate: 20.0, discountPercent: 10.0);

        // Assert
        $this->assertSame(10800, $result); // (10000 - 10%) * 1.20
    }

    public function test_zero_amount_returns_zero(): void
    {
        // Arrange
        $calculator = new PriceCalculator();

        // Act
        $result = $calculator->totalWithTax(amount: 0, taxRate: 20.0);

        // Assert
        $this->assertSame(0, $result);
    }
}
```
</content>
