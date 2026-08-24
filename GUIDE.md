# Deli Template — Laravel Implementation Guide

This folder is a mobile-focused HTML/CSS/JavaScript prototype. The data is currently hardcoded and the forms are display placeholders. Replace the placeholder values with Laravel routes, controllers, validation, and database records.

## Screens

| Template | Suggested Laravel route | Purpose |
|---|---|---|
| `login.html` | `GET /login` | Login form |
| `way-check.html` | `GET /way-check` | Create a delivery/order |
| `history.html` | `GET /history` | List all orders |
| `history-detail.html` | `GET /history/{order}` | Show one order |
| `shops.html` | `GET /shops` | List and create shops |
| `bikers.html` | `GET /bikers` | List and create bikers |
| `users.html` | `GET /users` | List and create users |

Rename the files to Blade views, for example `resources/views/orders/way-check.blade.php`.

## Recommended database structure

Use one `users` table for accounts and distinguish account types with a `role` field:

```text
users
- id
- name
- email (unique)
- password
- address (nullable)
- phone (nullable)
- role: shop | user | biker
- timestamps
```

For shops and bikers, either use profile tables or keep the prototype’s simple structure:

```text
shops
- id
- user_id
- name
- timestamps

bikers
- id
- user_id
- bike_number (nullable)
- timestamps
```

Orders should be represented separately:

```text
orders
- id
- shop_id
- biker_id (nullable)
- status: pending | delivered | failed
- customer_name
- customer_phone
- customer_address
- amount
- photo_path (nullable)
- delivered_at (nullable)
- timestamps
```

## Routes

Example `routes/web.php`:

```php
Route::middleware('auth')->group(function () {
    Route::get('/way-check', [OrderController::class, 'create'])->name('orders.create');
    Route::post('/way-check', [OrderController::class, 'store'])->name('orders.store');

    Route::get('/history', [OrderController::class, 'index'])->name('orders.index');
    Route::get('/history/{order}', [OrderController::class, 'show'])->name('orders.show');

    Route::resource('shops', ShopController::class)->only(['index', 'store']);
    Route::resource('bikers', BikerController::class)->only(['index', 'store']);
    Route::resource('users', UserController::class)->only(['index', 'store']);
});
```

Use Laravel’s built-in authentication routes or Fortify/Breeze for `login.html`. Do not expose the internal app drawer on the login screen.

## Blade conversion

Create a shared layout so the navbar and assets are not duplicated:

```blade
{{-- resources/views/layouts/app.blade.php --}}
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" href="{{ asset('css/global.css') }}">
    <link rel="stylesheet" href="{{ asset('css/components.css') }}">
    <link rel="stylesheet" href="{{ asset('css/screens.css') }}">
</head>
<body class="app-bg">
    @include('partials.navbar')
    @yield('content')
    @stack('scripts')
</body>
</html>
```

Move `css/` and `assets/` into `public/`, or compile them with Vite. The dummy package image should be available as `public/assets/logo.jpg`; uploaded order photos should use Laravel’s storage disk instead.

## Order create form

Convert the Way Check form into a POST form:

```blade
<form method="POST" action="{{ route('orders.store') }}" enctype="multipart/form-data">
    @csrf
    <select name="shop_id" required>
        @foreach ($shops as $shop)
            <option value="{{ $shop->id }}">{{ $shop->name }}</option>
        @endforeach
    </select>

    <select name="biker_id">
        @foreach ($bikers as $biker)
            <option value="{{ $biker->id }}">{{ $biker->user->name }}</option>
        @endforeach
    </select>

    <select name="status" required>
        <option value="pending">Pending</option>
        <option value="delivered">Delivered</option>
        <option value="failed">Failed</option>
    </select>

    <input name="customer_name" required>
    <input name="customer_phone" required>
    <input name="customer_address" required>
    <input name="amount" type="number" min="0" step="1" required>
    <input name="photo" type="file" accept="image/*">
    <button type="submit">Add way</button>
</form>
```

In `OrderController@store`, validate and store the photo:

```php
$data = $request->validate([
    'shop_id' => ['required', 'exists:shops,id'],
    'biker_id' => ['nullable', 'exists:bikers,id'],
    'status' => ['required', Rule::in(['pending', 'delivered', 'failed'])],
    'customer_name' => ['required', 'string', 'max:255'],
    'customer_phone' => ['required', 'string', 'max:30'],
    'customer_address' => ['required', 'string', 'max:1000'],
    'amount' => ['required', 'numeric', 'min:0'],
    'photo' => ['nullable', 'image', 'max:5120'],
]);

if ($request->hasFile('photo')) {
    $data['photo_path'] = $request->file('photo')->store('order-photos', 'public');
}

Order::create($data);
return redirect()->route('orders.index')->with('success', 'Order added.');
```

Run `php artisan storage:link` so public uploaded photos can be displayed with `Storage::url($order->photo_path)`.

## History list and detail

The history screen should query real orders instead of the hardcoded rows:

```php
$orders = Order::latest()->paginate(20);
return view('orders.history', compact('orders'));
```

In Blade, replace each dummy row with:

```blade
@foreach ($orders as $order)
    <tr>
        <td>{{ $loop->iteration }}</td>
        <td>{{ $order->customer_name }}</td>
        <td>{{ $order->created_at->format('d-m-Y') }}</td>
        <td><a class="table-action" href="{{ route('orders.show', $order) }}">View</a></td>
    </tr>
@endforeach
```

On the detail screen, use `$order->shop`, `$order->biker->user`, and the order fields. For the photo, use the uploaded path with a fallback:

```blade
<img src="{{ $order->photo_path ? Storage::url($order->photo_path) : asset('assets/logo.jpg') }}" alt="Package photo">
```

Escape all displayed values with normal Blade `{{ }}` output.

## Shops, bikers, and users

Each respective screen has an “Add new” action and its own form. Keep the forms separate at the route/controller level:

- Shops: create a user with `role = shop`, then create the related shop profile.
- Bikers: create a user with `role = biker`, then create the biker profile and optional bike number.
- Users: create a user with `role = user`.

Use `Hash::make($request->password)` when creating passwords. Never store plain-text passwords.

## Validation and security checklist

- Add `@csrf` to every POST form.
- Protect all app screens with `auth` middleware.
- Authorize shop, biker, and user management actions by role/policy.
- Validate uploaded files as images and limit file size.
- Use mass-assignment `$fillable` or explicit model assignment.
- Paginate history records.
- Return validation errors beside each field.
- Keep the current client-side preview only as a convenience; server-side validation remains authoritative.

## Front-end behavior to retain

- The sidebar drawer opens from the left and closes through the overlay.
- The Way Check photo field accepts one image and previews it locally.
- Buttons currently provide visual placeholder interactions; connect them to real POST/delete/edit actions as those routes are implemented.
- The current custom select script can remain for mobile presentation, but submitted values must come from named form controls.
