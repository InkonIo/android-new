# Технический отчёт по проекту "Skrepta"

## Android-приложение для маркетплейса

**Дата:** Декабрь 2025  
**Платформа:** Android (Kotlin + Jetpack Compose)  
**База данных:** Firebase Firestore + Firebase Authentication

---

## Команда разработки

| Участник | Роль | Зона ответственности |
|----------|------|---------------------|
| **Дидар** | Backend-разработчик | Логика работы с данными, ViewModel, бизнес-логика |
| **Алихан** | Backend-разработчик | Firebase интеграция, CRUD операции, аутентификация |
| **Наталья** | UI/UX дизайнер | Визуальное оформление, экраны, компоненты, анимации |
| **Рома** | Database Engineer | Настройка Firebase, репозитории, структура данных |

---

## Архитектура приложения

Приложение построено по архитектуре **MVVM** (Model-View-ViewModel):

```
┌─────────────────────────────────────────────────────────────┐
│                         VIEW                                 │
│  (Screens: Home, Category, Feed, Auth, Favorites, Cart)     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      VIEWMODEL                               │
│              (SkreptaViewModel.kt)                          │
│    - Хранит состояние UI                                    │
│    - Обрабатывает действия пользователя                     │
│    - Связывает View и Model                                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        MODEL                                 │
│  - FirebaseRepository.kt (работа с Firestore)               │
│  - SkreptaRepository.kt (локальные данные)                  │
│  - Data classes (Category, FeedItem, CartItem)              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   FIREBASE BACKEND                           │
│         Firestore Database + Authentication                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Структура проекта

```
com.example.skrepta/
│
├── model/                          # Слой данных
│   ├── Category.kt                 # Модель категории
│   ├── FeedItem.kt                 # Модель товара в ленте
│   ├── CartItem.kt                 # Модель элемента корзины
│   ├── Repositories.kt             # Локальные данные (mock)
│   └── FirebaseRepository.kt       # Работа с Firebase
│
├── viewmodel/                      # Слой бизнес-логики
│   └── SkreptaViewModel.kt         # Главный ViewModel
│
└── view/                           # Слой представления
    ├── MainActivity.kt             # Точка входа
    ├── SkreptaApp.kt              # Навигация и тема
    ├── components/                 # UI компоненты
    │   ├── Cards.kt               # Карточки товаров
    │   ├── GridItems.kt           # Сетка товаров
    │   └── WaveFooter.kt          # Анимированный футер
    └── screens/                    # Экраны
        ├── HomeScreen.kt          # Главный экран
        ├── CategoryScreen.kt      # Экран категории
        ├── FeedScreen.kt          # Лента товаров
        ├── AuthScreen.kt          # Авторизация
        ├── FavoritesScreen.kt     # Избранное
        └── CartScreen.kt          # Корзина
```

---

# ЧАСТЬ 1: BACKEND (Дидар, Алихан)

## 1.1 SkreptaViewModel.kt

**Автор:** Дидар  
**Расположение:** `viewmodel/SkreptaViewModel.kt`  
**Назначение:** Центральный компонент бизнес-логики, связывает UI с данными

### Состояния (State)

```kotlin
// Категории товаров
private val _categories = MutableStateFlow<List<Category>>(emptyList())
val categories: StateFlow<List<Category>> = _categories

// Лента товаров
private val _feed = MutableStateFlow<List<FeedItem>>(emptyList())
val feed: StateFlow<List<FeedItem>> = _feed

// Избранное (Set из ID товаров)
private val _favorites = MutableStateFlow<Set<String>>(emptySet())
val favorites: StateFlow<Set<String>> = _favorites

// Корзина
private val _cartItems = MutableStateFlow<List<CartItem>>(emptyList())
val cartItems: StateFlow<List<CartItem>> = _cartItems

// Счётчик товаров в корзине
private val _cartCount = MutableStateFlow(0)
val cartCount: StateFlow<Int> = _cartCount

// Статус авторизации
private val _isLoggedIn = MutableStateFlow(false)
val isLoggedIn: StateFlow<Boolean> = _isLoggedIn

// Email текущего пользователя
private val _currentUserEmail = MutableStateFlow<String?>(null)
val currentUserEmail: StateFlow<String?> = _currentUserEmail
```

### Методы

| Метод | Описание | Используется в |
|-------|----------|----------------|
| `loadLocalData()` | Загружает категории и товары из локального репозитория | При инициализации |
| `observeAuth()` | Слушает изменения состояния авторизации Firebase | При инициализации |
| `observeFavorites()` | Подписывается на изменения избранного в Firestore | После входа |
| `observeCart()` | Подписывается на изменения корзины в Firestore | После входа |
| `toggleFavorite(productId)` | Добавляет/удаляет товар из избранного | HomeScreen, CategoryScreen, FeedScreen |
| `addToCart(productId)` | Добавляет товар в корзину | HomeScreen, CategoryScreen, FeedScreen, FavoritesScreen |
| `removeFromCart(productId)` | Удаляет товар из корзины | CartScreen |
| `updateCartQuantity(productId, qty)` | Изменяет количество товара | CartScreen |
| `clearCart()` | Очищает всю корзину | CartScreen |
| `signOut()` | Выход из аккаунта | HomeScreen |

### Пример использования в UI:

```kotlin
// В HomeScreen.kt
val favorites by vm.favorites.collectAsState()
val isLoggedIn by vm.isLoggedIn.collectAsState()

// При нажатии на сердечко
IconButton(onClick = { 
    if (isLoggedIn) vm.toggleFavorite(item.id.toString()) 
    else onOpenAuth() 
})
```

---

## 1.2 FirebaseRepository.kt

**Автор:** Алихан  
**Расположение:** `model/FirebaseRepository.kt`  
**Назначение:** CRUD операции с Firebase Firestore

### CRUD операции для Избранного

#### CREATE — Добавление в избранное
```kotlin
suspend fun addToFavorites(productId: String): Boolean {
    val userId = currentUserId ?: return false
    
    return try {
        val data = hashMapOf(
            "productId" to productId,
            "addedAt" to System.currentTimeMillis()
        )
        
        firestore
            .collection("users")
            .document(userId)
            .collection("favorites")
            .document(productId)
            .set(data)
            .await()
        
        true
    } catch (e: Exception) {
        e.printStackTrace()
        false
    }
}
```
**Что делает:** Создаёт документ в коллекции `favorites` пользователя  
**Структура в Firestore:** `users/{userId}/favorites/{productId}`

#### READ — Чтение избранного (Realtime)
```kotlin
fun getFavoritesFlow(): Flow<List<String>> = callbackFlow {
    val userId = currentUserId
    if (userId == null) {
        trySend(emptyList())
        close()
        return@callbackFlow
    }
    
    val registration = firestore
        .collection("users")
        .document(userId)
        .collection("favorites")
        .addSnapshotListener { snapshot, error ->
            if (error != null) {
                trySend(emptyList())
                return@addSnapshotListener
            }
            
            val ids = mutableListOf<String>()
            snapshot?.documents?.forEach { doc ->
                doc.getString("productId")?.let { ids.add(it) }
            }
            trySend(ids)
        }
    
    awaitClose { registration.remove() }
}
```
**Что делает:** Возвращает Flow со списком ID избранных товаров, обновляется в реальном времени  
**Используется в:** `SkreptaViewModel.observeFavorites()`

#### DELETE — Удаление из избранного
```kotlin
suspend fun removeFromFavorites(productId: String): Boolean {
    val userId = currentUserId ?: return false
    
    return try {
        firestore
            .collection("users")
            .document(userId)
            .collection("favorites")
            .document(productId)
            .delete()
            .await()
        
        true
    } catch (e: Exception) {
        e.printStackTrace()
        false
    }
}
```
**Что делает:** Удаляет документ из коллекции `favorites`

---

### CRUD операции для Корзины

#### CREATE — Добавление в корзину
```kotlin
suspend fun addToCart(productId: String): Boolean {
    val userId = currentUserId ?: return false
    
    return try {
        val docRef = firestore
            .collection("users")
            .document(userId)
            .collection("cart")
            .document(productId)
        
        val doc = docRef.get().await()
        
        if (doc.exists()) {
            // Товар уже есть — увеличиваем количество
            val currentQty = doc.getLong("quantity")?.toInt() ?: 1
            docRef.update("quantity", currentQty + 1).await()
        } else {
            // Товара нет — создаём новый
            val data = hashMapOf(
                "productId" to productId,
                "quantity" to 1,
                "addedAt" to System.currentTimeMillis()
            )
            docRef.set(data).await()
        }
        
        true
    } catch (e: Exception) {
        e.printStackTrace()
        false
    }
}
```
**Что делает:** Добавляет товар в корзину. Если товар уже есть — увеличивает количество на 1  
**Структура в Firestore:** `users/{userId}/cart/{productId}`

#### READ — Чтение корзины (Realtime)
```kotlin
fun getCartFlow(): Flow<List<CartItem>> = callbackFlow {
    val userId = currentUserId
    if (userId == null) {
        trySend(emptyList())
        close()
        return@callbackFlow
    }
    
    val registration = firestore
        .collection("users")
        .document(userId)
        .collection("cart")
        .addSnapshotListener { snapshot, error ->
            if (error != null) {
                trySend(emptyList())
                return@addSnapshotListener
            }
            
            val items = mutableListOf<CartItem>()
            snapshot?.documents?.forEach { doc ->
                val productId = doc.getString("productId") ?: return@forEach
                val quantity = doc.getLong("quantity")?.toInt() ?: 1
                val addedAt = doc.getLong("addedAt") ?: 0L
                
                items.add(
                    CartItem(
                        id = doc.id,
                        productId = productId,
                        userId = userId,
                        quantity = quantity,
                        addedAt = addedAt
                    )
                )
            }
            trySend(items)
        }
    
    awaitClose { registration.remove() }
}
```
**Что делает:** Возвращает Flow со списком товаров в корзине, обновляется в реальном времени

#### UPDATE — Изменение количества
```kotlin
suspend fun updateCartQuantity(productId: String, quantity: Int): Boolean {
    val userId = currentUserId ?: return false
    
    return try {
        if (quantity <= 0) {
            removeFromCart(productId)
        } else {
            firestore
                .collection("users")
                .document(userId)
                .collection("cart")
                .document(productId)
                .update("quantity", quantity)
                .await()
            true
        }
    } catch (e: Exception) {
        e.printStackTrace()
        false
    }
}
```
**Что делает:** Обновляет поле `quantity`. Если количество ≤ 0 — удаляет товар

#### DELETE — Удаление из корзины
```kotlin
suspend fun removeFromCart(productId: String): Boolean {
    val userId = currentUserId ?: return false
    
    return try {
        firestore
            .collection("users")
            .document(userId)
            .collection("cart")
            .document(productId)
            .delete()
            .await()
        
        true
    } catch (e: Exception) {
        e.printStackTrace()
        false
    }
}
```

#### DELETE ALL — Очистка корзины
```kotlin
suspend fun clearCart(): Boolean {
    val userId = currentUserId ?: return false
    
    return try {
        val snapshot = firestore
            .collection("users")
            .document(userId)
            .collection("cart")
            .get()
            .await()
        
        for (doc in snapshot.documents) {
            doc.reference.delete().await()
        }
        
        true
    } catch (e: Exception) {
        e.printStackTrace()
        false
    }
}
```
**Что делает:** Удаляет все документы из коллекции `cart` пользователя

---

## 1.3 Структура данных в Firebase

**Автор:** Рома  
**База данных:** Firebase Firestore

### Схема базы данных

```
firestore/
└── users/                              # Коллекция пользователей
    └── {userId}/                       # Документ пользователя (UID из Auth)
        │
        ├── favorites/                  # Подколлекция избранного
        │   ├── {productId}/           # Документ = ID товара
        │   │   ├── productId: "1"     # ID товара (string)
        │   │   └── addedAt: 173...    # Timestamp добавления (long)
        │   │
        │   └── {productId}/
        │       ├── productId: "3"
        │       └── addedAt: 173...
        │
        └── cart/                       # Подколлекция корзины
            ├── {productId}/           # Документ = ID товара
            │   ├── productId: "1"     # ID товара (string)
            │   ├── quantity: 2        # Количество (int)
            │   └── addedAt: 173...    # Timestamp добавления (long)
            │
            └── {productId}/
                ├── productId: "5"
                ├── quantity: 1
                └── addedAt: 173...
```

### Модели данных

#### CartItem.kt
```kotlin
data class CartItem(
    val id: String = "",           // ID документа в Firestore
    val productId: String = "",    // ID товара
    val userId: String = "",       // ID пользователя
    val quantity: Int = 1,         // Количество
    val addedAt: Long = System.currentTimeMillis()  // Время добавления
)
```

#### Category.kt
```kotlin
data class Category(
    val id: Int,                   // ID категории
    val title: String,             // Название ("Женский уход")
    val imageRes: Int              // Ресурс изображения (R.drawable.cat_women)
)
```

#### FeedItem.kt
```kotlin
data class FeedItem(
    val id: Int,                   // ID товара
    val imageUrl: String? = null,  // URL изображения (из интернета)
    val imageRes: Int? = null,     // Локальный ресурс изображения
    val link: String? = null       // Ссылка на внешний сайт
)
```

---

# ЧАСТЬ 2: FRONTEND / UI (Наталья)

## 2.1 Навигация — SkreptaApp.kt

**Расположение:** `view/SkreptaApp.kt`  
**Назначение:** Настройка навигации и темы приложения

### Маршруты (Routes)

```kotlin
enum class Route {
    Home,       // Главный экран
    Category,   // Экран категории
    Feed,       // Лента товаров
    Auth,       // Авторизация/регистрация
    Favorites,  // Избранное
    Cart        // Корзина
}
```

### Цветовая схема

```kotlin
private val SkreptaColors = lightColorScheme(
    primary = Color(0xFFdf1778),           // Основной розовый
    onPrimary = Color.White,               // Текст на розовом
    primaryContainer = Color(0xFFFFE5EE),  // Светло-розовый контейнер
    secondary = Color(0xFFFF6B9D),         // Вторичный розовый
    background = Color(0xFFFFF5F8),        // Фон приложения
    surface = Color.White,                 // Поверхности (карточки)
    onBackground = Color(0xFF1A1A2E),      // Текст на фоне
    onSurface = Color(0xFF1A1A2E)          // Текст на поверхностях
)
```

### Граф навигации

```kotlin
NavHost(navController = nav, startDestination = Route.Home.name) {
    
    composable(Route.Home.name) {
        HomeScreen(
            vm = vm,
            onOpenCategory = { nav.navigate(Route.Category.name) },
            onOpenFeed = { nav.navigate(Route.Feed.name) },
            onOpenAuth = { nav.navigate(Route.Auth.name) },
            onOpenFavorites = { nav.navigate(Route.Favorites.name) },
            onOpenCart = { nav.navigate(Route.Cart.name) }
        )
    }
    
    composable(Route.Category.name) {
        CategoryScreen(onBack = { nav.popBackStack() }, vm = vm)
    }
    
    composable(Route.Feed.name) {
        FeedScreen(onBack = { nav.popBackStack() }, vm = vm)
    }
    
    composable(Route.Auth.name) {
        AuthScreen(onBack = { nav.popBackStack() })
    }
    
    composable(Route.Favorites.name) {
        FavoritesScreen(
            onBack = { nav.popBackStack() },
            onOpenAuth = { nav.navigate(Route.Auth.name) },
            vm = vm
        )
    }
    
    composable(Route.Cart.name) {
        CartScreen(
            onBack = { nav.popBackStack() },
            onOpenAuth = { nav.navigate(Route.Auth.name) },
            vm = vm
        )
    }
}
```

---

## 2.2 Главный экран — HomeScreen.kt

**Расположение:** `view/screens/HomeScreen.kt`  
**Назначение:** Главная страница приложения

### Структура экрана

```
┌─────────────────────────────────────┐
│  [S] Skrepta     ♡  🛒  👤         │  ← TopAppBar
├─────────────────────────────────────┤
│  ┌─────────────────────────────┐   │
│  │      БАННЕР (слайдер)       │   │  ← BannerSlider
│  │         ● ○                 │   │
│  └─────────────────────────────┘   │
│                                     │
│  👤 Добро пожаловать!              │  ← WelcomeCard (если авторизован)
│     user@email.com                  │
│                                     │
│  Основные категории          Все → │  ← SectionHeader
│  ┌───┐ ┌───┐ ┌───┐                │
│  │   │ │   │ │   │                 │  ← CategoryGrid
│  └───┘ └───┘ └───┘                 │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ Войдите в аккаунт   [Войти] │   │  ← AuthPromptCard (если не авторизован)
│  └─────────────────────────────┘   │
│                                     │
│  Лента                       Все → │  ← SectionHeader
│  ┌───┐ ┌───┐                       │
│  │ ♡ │ │ ♡ │                       │  ← FeedCardWithFavorite
│  │ + │ │ + │                       │
│  └───┘ └───┘                       │
│                                     │
│  ～～～～～～～～～～～～～～～～   │  ← WaveFooter
│  Главная  Категории  Лента         │
│  ©2025 Skrepta                      │
└─────────────────────────────────────┘
```

### Ключевые компоненты

#### TopAppBar с бейджами
```kotlin
TopAppBar(
    title = {
        Row(verticalAlignment = Alignment.CenterVertically) {
            // Логотип
            Box(
                modifier = Modifier
                    .size(36.dp)
                    .clip(RoundedCornerShape(10.dp))
                    .background(Brush.linearGradient(listOf(PrimaryPink, LightPink))),
                contentAlignment = Alignment.Center
            ) {
                Text("S", fontSize = 20.sp, fontWeight = FontWeight.Bold, color = Color.White)
            }
            Spacer(modifier = Modifier.width(10.dp))
            Text("Skrepta", fontWeight = FontWeight.Bold, fontSize = 22.sp)
        }
    },
    actions = {
        // Избранное с бейджем
        IconButton(onClick = onOpenFavorites) {
            BadgedBox(
                badge = {
                    if (favorites.isNotEmpty()) {
                        Badge(containerColor = PrimaryPink) {
                            Text(favorites.size.toString())
                        }
                    }
                }
            ) {
                Icon(
                    if (favorites.isNotEmpty()) Icons.Default.Favorite 
                    else Icons.Outlined.FavoriteBorder,
                    tint = if (favorites.isNotEmpty()) PrimaryPink else Color.Gray
                )
            }
        }
        
        // Корзина с бейджем
        IconButton(onClick = onOpenCart) {
            BadgedBox(
                badge = {
                    if (cartCount > 0) {
                        Badge(containerColor = PrimaryPink) {
                            Text(cartCount.toString())
                        }
                    }
                }
            ) {
                Icon(Icons.Default.ShoppingCart)
            }
        }
    }
)
```

#### Карточка товара с избранным
```kotlin
@Composable
fun FeedCardWithFavorite(
    item: FeedItem,
    isFavorite: Boolean,
    isLoggedIn: Boolean,
    onToggleFavorite: () -> Unit,
    onAddToCart: () -> Unit,
    onOpenAuth: () -> Unit
) {
    // Анимация сердечка при добавлении
    val scale by animateFloatAsState(
        targetValue = if (isFavorite) 1.2f else 1f,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy,
            stiffness = Spring.StiffnessLow
        )
    )
    
    Card(...) {
        Box {
            // Изображение товара
            Image(...)
            
            // Кнопка избранного (сердечко)
            IconButton(
                onClick = { 
                    if (isLoggedIn) onToggleFavorite() 
                    else onOpenAuth()  // Если не авторизован — на экран входа
                },
                modifier = Modifier
                    .align(Alignment.TopEnd)
                    .clip(CircleShape)
                    .background(Color.White.copy(alpha = 0.9f))
            ) {
                Icon(
                    if (isFavorite) Icons.Default.Favorite 
                    else Icons.Outlined.FavoriteBorder,
                    tint = if (isFavorite) PrimaryPink else Color.Gray,
                    modifier = Modifier.scale(scale)  // Анимация
                )
            }
            
            // Кнопка "В корзину"
            IconButton(
                onClick = { 
                    if (isLoggedIn) onAddToCart() 
                    else onOpenAuth() 
                },
                modifier = Modifier
                    .align(Alignment.BottomEnd)
                    .clip(CircleShape)
                    .background(PrimaryPink)
            ) {
                Icon(Icons.Default.Add, tint = Color.White)
            }
        }
    }
}
```

---

## 2.3 Экран авторизации — AuthScreen.kt

**Расположение:** `view/screens/AuthScreen.kt`  
**Назначение:** Вход и регистрация пользователей

### Особенности дизайна

- **Градиентный фон** с размытыми декоративными кругами
- **Пульсирующий логотип** "S"
- **Переключатель** между входом и регистрацией
- **Показать/скрыть пароль**
- **Анимация появления** при загрузке экрана

### Логика авторизации

```kotlin
// Вход
auth.signInWithEmailAndPassword(email, pass)
    .addOnSuccessListener {
        Toast.makeText(ctx, "Вход выполнен!", Toast.LENGTH_SHORT).show()
        onBack()  // Возврат на предыдущий экран
    }
    .addOnFailureListener { e ->
        Toast.makeText(ctx, "Ошибка: ${e.message}", Toast.LENGTH_LONG).show()
    }
    .addOnCompleteListener { 
        isLoading = false 
    }

// Регистрация
auth.createUserWithEmailAndPassword(email, pass)
    .addOnSuccessListener {
        Toast.makeText(ctx, "Регистрация успешна!", Toast.LENGTH_SHORT).show()
        onBack()
    }
    .addOnFailureListener { e ->
        Toast.makeText(ctx, "Ошибка: ${e.message}", Toast.LENGTH_LONG).show()
    }
```

---

## 2.4 Экран избранного — FavoritesScreen.kt

**Расположение:** `view/screens/FavoritesScreen.kt`  
**Назначение:** Отображение списка избранных товаров

### Состояния экрана

1. **Не авторизован** → Показывает приглашение войти
2. **Пустое избранное** → Анимированное сердечко + текст
3. **Есть товары** → Список карточек

### Карточка избранного товара

```kotlin
@Composable
private fun FavoriteItemCard(
    item: FeedItem,
    onRemove: () -> Unit,      // Удалить из избранного
    onAddToCart: () -> Unit    // Добавить в корзину
) {
    Card(...) {
        Row(...) {
            // Изображение товара
            Box(modifier = Modifier.size(80.dp)) {
                Image(...)
            }
            
            // Информация
            Column(modifier = Modifier.weight(1f)) {
                Text("Товар #${item.id}")
                Text("В наличии", color = Color.Green)
            }
            
            // Кнопки действий
            Column {
                // Удалить из избранного (розовое сердечко)
                IconButton(onClick = onRemove) {
                    Icon(Icons.Default.Favorite, tint = PrimaryPink)
                }
                
                // Добавить в корзину
                IconButton(onClick = onAddToCart) {
                    Icon(Icons.Default.ShoppingCart, tint = Color.White)
                }
            }
        }
    }
}
```

---

## 2.5 Экран корзины — CartScreen.kt

**Расположение:** `view/screens/CartScreen.kt`  
**Назначение:** Управление корзиной покупок

### Функционал

- Просмотр товаров в корзине
- Изменение количества (+/-)
- Удаление товаров
- Очистка всей корзины
- Панель оформления заказа

### Управление количеством

```kotlin
Row(verticalAlignment = Alignment.CenterVertically) {
    // Кнопка "-"
    IconButton(
        onClick = { 
            if (cartItem.quantity > 1) 
                onUpdateQuantity(cartItem.quantity - 1)  // Уменьшить
            else 
                onRemove()  // Если 1 — удалить
        }
    ) {
        Icon(Icons.Default.Remove)
    }
    
    // Количество
    Text(
        text = cartItem.quantity.toString(),
        fontWeight = FontWeight.Bold
    )
    
    // Кнопка "+"
    IconButton(
        onClick = { onUpdateQuantity(cartItem.quantity + 1) }
    ) {
        Icon(Icons.Default.Add)
    }
}
```

### Панель оформления

```kotlin
@Composable
private fun CheckoutPanel(itemCount: Int) {
    Surface(shadowElevation = 16.dp, color = Color.White) {
        Row(
            modifier = Modifier.fillMaxWidth().padding(16.dp),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Column {
                Text("Итого", color = Color.Gray)
                Text("$itemCount ${getItemsText(itemCount)}", fontWeight = FontWeight.Bold)
            }
            
            Button(onClick = { /* TODO: Оформление заказа */ }) {
                Text("Оформить заказ")
            }
        }
    }
}

// Склонение слова "товар"
private fun getItemsText(count: Int) = when {
    count % 100 in 11..19 -> "товаров"
    count % 10 == 1 -> "товар"
    count % 10 in 2..4 -> "товара"
    else -> "товаров"
}
```

---

## 2.6 Компоненты — WaveFooter.kt

**Расположение:** `view/components/WaveFooter.kt`  
**Назначение:** Анимированный волнообразный футер

### Анимация волн

```kotlin
// 4 слоя волн с разной скоростью
val p1 by rememberInfiniteTransition()
    .animateFloat(0f, 1f, infiniteRepeatable(tween(4000)))
val p2 by rememberInfiniteTransition()
    .animateFloat(0f, 1f, infiniteRepeatable(tween(4000)))
val p3 by rememberInfiniteTransition()
    .animateFloat(0f, 1f, infiniteRepeatable(tween(3000)))
val p4 by rememberInfiniteTransition()
    .animateFloat(0f, 1f, infiniteRepeatable(tween(3000)))

// Отрисовка слоёв с разной прозрачностью
WaveLayer(img, progress = p1, alpha = 1f, invert = false)
WaveLayer(img, progress = p2, alpha = 0.5f, invert = true)
WaveLayer(img, progress = p3, alpha = 0.2f, invert = false)
WaveLayer(img, progress = p4, alpha = 0.7f, invert = true)
```

---

# ЧАСТЬ 3: НАСТРОЙКА FIREBASE (Рома)

## 3.1 Firebase Authentication

**Тип:** Email/Password  
**Настройка:**
1. Firebase Console → Authentication → Sign-in method
2. Включить Email/Password provider

### Использование в коде

```kotlin
// Получение экземпляра
private val auth = FirebaseAuth.getInstance()

// Текущий пользователь
val currentUser = auth.currentUser
val userId = currentUser?.uid
val email = currentUser?.email

// Слушатель изменений авторизации
auth.addAuthStateListener { firebaseAuth ->
    val user = firebaseAuth.currentUser
    _isLoggedIn.value = user != null
    _currentUserEmail.value = user?.email
}

// Выход
auth.signOut()
```

## 3.2 Firebase Firestore

**Режим:** Test mode (для разработки)  
**Правила безопасности:**
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Пользователь может читать/писать только свои данные
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

## 3.3 Зависимости (build.gradle.kts)

```kotlin
dependencies {
    // Firebase BOM (Bill of Materials)
    implementation(platform("com.google.firebase:firebase-bom:32.7.0"))
    
    // Firebase Auth
    implementation("com.google.firebase:firebase-auth-ktx")
    
    // Firebase Firestore
    implementation("com.google.firebase:firebase-firestore-ktx")
}
```

## 3.4 Плагин Google Services

**Проектный build.gradle.kts:**
```kotlin
plugins {
    id("com.google.gms.google-services") version "4.4.0" apply false
}
```

**App build.gradle.kts:**
```kotlin
plugins {
    id("com.google.gms.google-services")
}
```

**Файл конфигурации:** `app/google-services.json` (скачать из Firebase Console)

---

# ЗАКЛЮЧЕНИЕ

## Реализованный функционал

| Функция | Статус | Описание |
|---------|--------|----------|
| Авторизация | ✅ | Email/Password через Firebase Auth |
| Регистрация | ✅ | Создание нового аккаунта |
| Просмотр категорий | ✅ | Сетка категорий товаров |
| Лента товаров | ✅ | Сетка/список товаров |
| Избранное | ✅ | CRUD с синхронизацией в Firestore |
| Корзина | ✅ | CRUD с управлением количеством |
| Анимации | ✅ | Волны, сердечки, появление элементов |

## Технологический стек

- **Язык:** Kotlin 2.0.21
- **UI:** Jetpack Compose + Material 3
- **Архитектура:** MVVM
- **DI:** Manual (можно добавить Hilt)
- **Backend:** Firebase (Auth + Firestore)
- **Навигация:** Navigation Compose
- **Изображения:** Coil
- **Асинхронность:** Kotlin Coroutines + Flow

## Дальнейшее развитие

1. Добавить поиск товаров
2. Реализовать оформление заказа
3. Добавить push-уведомления (Firebase Cloud Messaging)
4. Реализовать профиль пользователя
5. Добавить фильтры и сортировку
6. Интегрировать оплату

---

**Отчёт подготовлен командой разработки Skrepta**  
**Декабрь 2025**
