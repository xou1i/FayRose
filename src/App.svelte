<script>
  let token = '';
  let user = null;
  let currentCategory = 'Flowers';
  let selectedProduct = null;
  let quantity = 1;
  let wrappingType = '';
  let cardStyle = '';
  let messageText = '';
  let loadingAuth = false;
  let loadingPay = false;
  let errorMessage = '';
  let showCustomize = false;

  const categories = ['Flowers', 'Gifts', 'Wrapping', 'Cards'];

  const products = {
    Flowers: [
      {
        id: 1,
        name: 'Rose Bouquet',
        price: 25000,
        image: 'https://via.placeholder.com/300x300.png?text=Rose+Bouquet'
      },
      {
        id: 2,
        name: 'Tulip Bouquet',
        price: 22000,
        image: 'https://via.placeholder.com/300x300.png?text=Tulip+Bouquet'
      }
    ],
    Gifts: [
      {
        id: 3,
        name: 'Gift Box',
        price: 15000,
        image: 'https://via.placeholder.com/300x300.png?text=Gift+Box'
      },
      {
        id: 4,
        name: 'Teddy Bear',
        price: 18000,
        image: 'https://via.placeholder.com/300x300.png?text=Teddy+Bear'
      }
    ],
    Wrapping: [
      {
        id: 5,
        name: 'Luxury Wrapping',
        price: 5000,
        image: 'https://via.placeholder.com/300x300.png?text=Luxury+Wrapping'
      },
      {
        id: 6,
        name: 'Classic Wrapping',
        price: 3000,
        image: 'https://via.placeholder.com/300x300.png?text=Classic+Wrapping'
      }
    ],
    Cards: [
      {
        id: 7,
        name: 'Message Card',
        price: 2000,
        image: 'https://via.placeholder.com/300x300.png?text=Message+Card'
      },
      {
        id: 8,
        name: 'Love Card',
        price: 2500,
        image: 'https://via.placeholder.com/300x300.png?text=Love+Card'
      }
    ]
  };

  const wrappingOptions = ['Standard', 'Premium', 'Luxury'];
  const cardOptions = ['Classic', 'Modern', 'Romantic'];

  $: currentProducts = products[currentCategory] || [];
  $: wrappingPrice = wrappingType === 'Premium' ? 3000 : wrappingType === 'Luxury' ? 5000 : 0;
  $: cardPrice = cardStyle === 'Modern' ? 500 : cardStyle === 'Romantic' ? 1000 : 0;
  $: finalPrice = selectedProduct ? (selectedProduct.price * quantity) + wrappingPrice + cardPrice : 0;
  $: canPay = token && user && quantity > 0 && !loadingPay && selectedProduct;

  function loadStoredData() {
    if (typeof my !== 'undefined' && my.getStorageSync) {
      try {
        const storedToken = my.getStorageSync({ key: 'fayrose_token' });
        const storedUser = my.getStorageSync({ key: 'fayrose_user' });
        if (storedToken && storedToken.data) {
          token = storedToken.data;
        }
        if (storedUser && storedUser.data) {
          user = JSON.parse(storedUser.data);
        }
      } catch (e) {
        console.error('Error loading storage:', e);
      }
    }
  }

  function saveToken(newToken) {
    token = newToken;
    if (typeof my !== 'undefined' && my.setStorageSync) {
      my.setStorageSync({
        key: 'fayrose_token',
        data: newToken
      });
    }
  }

  function saveUser(userData) {
    user = userData;
    if (typeof my !== 'undefined' && my.setStorageSync) {
      my.setStorageSync({
        key: 'fayrose_user',
        data: JSON.stringify(userData)
      });
    }
  }

  function handleAuth() {
    if (loadingAuth) return;
    loadingAuth = true;
    errorMessage = '';

    if (typeof my === 'undefined' || !my.getAuthCode) {
      errorMessage = 'SuperQi environment not detected';
      loadingAuth = false;
      return;
    }

    my.getAuthCode({
      scopes: ['auth_base', 'USER_ID'],
      success: (res) => {
        const authCode = res.authCode;
        fetch('https://its.mouamle.space/api/auth-with-superQi', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({ token: authCode })
        })
          .then(response => response.json())
          .then(data => {
            if (data.token && data.record) {
              saveToken(data.token);
              saveUser({ id: data.record.id });
              loadingAuth = false;
              errorMessage = '';
            } else {
              errorMessage = 'Authentication failed';
              loadingAuth = false;
            }
          })
          .catch(err => {
            errorMessage = 'Network error: ' + err.message;
            loadingAuth = false;
          });
      },
      fail: (err) => {
        errorMessage = 'Auth failed: ' + (err.errorMessage || 'Unknown error');
        loadingAuth = false;
      }
    });
  }

  function selectCategory(cat) {
    currentCategory = cat;
    showCustomize = false;
    selectedProduct = null;
  }

  function customizeProduct(product) {
    selectedProduct = product;
    quantity = 1;
    wrappingType = '';
    cardStyle = '';
    messageText = '';
    showCustomize = true;
  }

  function increaseQuantity() {
    quantity += 1;
  }

  function decreaseQuantity() {
    if (quantity > 1) {
      quantity -= 1;
    }
  }

  function handlePayment() {
    if (!canPay) return;
    loadingPay = true;
    errorMessage = '';

    fetch('https://its.mouamle.space/api/payment', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': token
      },
      body: JSON.stringify({
        productId: selectedProduct.id,
        quantity: quantity,
        wrapping: wrappingType,
        card: cardStyle,
        message: messageText,
        totalPrice: finalPrice
      })
    })
      .then(response => response.json())
      .then(data => {
        if (data.url) {
          if (typeof my !== 'undefined' && my.tradePay) {
            my.tradePay({
              paymentUrl: data.url,
              success: () => {
                if (typeof my !== 'undefined' && my.alert) {
                  my.alert({ content: 'Payment successful' });
                }
                loadingPay = false;
                showCustomize = false;
                selectedProduct = null;
              },
              fail: () => {
                if (typeof my !== 'undefined' && my.alert) {
                  my.alert({ content: 'Payment failed' });
                }
                loadingPay = false;
              }
            });
          } else {
            errorMessage = 'Payment UI not available';
            loadingPay = false;
          }
        } else {
          errorMessage = 'Payment URL not received';
          loadingPay = false;
        }
      })
      .catch(err => {
        errorMessage = 'Payment error: ' + err.message;
        loadingPay = false;
      });
  }

  function closeCustomize() {
    showCustomize = false;
    selectedProduct = null;
  }

  loadStoredData();
</script>

<svelte:window onload={loadStoredData} />

<div class="app">
  <nav class="navbar">
    <h1>FayRose</h1>
    {#if user}
      <div class="user-info">User ID: {user.id}</div>
    {/if}
  </nav>

  {#if !token}
    <div class="auth-section">
      <div class="glass-card auth-card">
        <h2>Welcome to FayRose</h2>
        <p>Please authenticate with SuperQi to continue</p>
        <button class="auth-btn" on:click={handleAuth} disabled={loadingAuth}>
          {loadingAuth ? 'Authenticating...' : 'Login with SuperQi'}
        </button>
        {#if errorMessage}
          <div class="error-message">{errorMessage}</div>
        {/if}
      </div>
    </div>
  {/if}

  {#if token && !showCustomize}
    <div class="category-bar">
      <div class="category-scroll">
        {#each categories as cat}
          <button
            class="category-btn"
            class:active={currentCategory === cat}
            on:click={() => selectCategory(cat)}
          >
            {cat}
          </button>
        {/each}
      </div>
    </div>

    <div class="products-section">
      <h2 class="section-title">{currentCategory}</h2>
      <div class="products-grid">
        {#each currentProducts as product}
          <div class="glass-card product-card">
            <img src={product.image} alt={product.name} />
            <h3>{product.name}</h3>
            <p class="price">{product.price.toLocaleString()} IQD</p>
            <button class="customize-btn" on:click={() => customizeProduct(product)}>
              Customize
            </button>
          </div>
        {/each}
      </div>
    </div>
  {/if}

  {#if showCustomize && selectedProduct}
    <div class="customize-overlay" on:click={closeCustomize}>
      <div class="customize-container" on:click|stopPropagation>
        <div class="glass-card customize-card">
          <button class="close-btn" on:click={closeCustomize}>×</button>
          <h2>Customize Your Order</h2>
          
          <div class="product-preview">
            <img src={selectedProduct.image} alt={selectedProduct.name} />
            <h3>{selectedProduct.name}</h3>
            <p class="base-price">Base Price: {selectedProduct.price.toLocaleString()} IQD</p>
          </div>

          <div class="customization-section">
            <div class="customization-item">
              <label>Quantity</label>
              <div class="quantity-control">
                <button class="qty-btn" on:click={decreaseQuantity}>−</button>
                <span class="qty-value">{quantity}</span>
                <button class="qty-btn" on:click={increaseQuantity}>+</button>
              </div>
            </div>

            <div class="customization-item">
              <label>Wrapping Type</label>
              <div class="option-buttons">
                {#each wrappingOptions as option}
                  <button
                    class="option-btn"
                    class:selected={wrappingType === option}
                    on:click={() => wrappingType = option}
                  >
                    {option}
                  </button>
                {/each}
              </div>
            </div>

            <div class="customization-item">
              <label>Card Style</label>
              <div class="option-buttons">
                {#each cardOptions as option}
                  <button
                    class="option-btn"
                    class:selected={cardStyle === option}
                    on:click={() => cardStyle = option}
                  >
                    {option}
                  </button>
                {/each}
              </div>
            </div>

            <div class="customization-item">
              <label>Custom Message</label>
              <textarea
                class="message-input"
                placeholder="Enter your message here..."
                bind:value={messageText}
                rows="4"
              ></textarea>
            </div>
          </div>

          <div class="order-summary">
            <h3>Order Summary</h3>
            <div class="summary-row">
              <span>Product:</span>
              <span>{selectedProduct.name}</span>
            </div>
            <div class="summary-row">
              <span>Quantity:</span>
              <span>{quantity}</span>
            </div>
            {#if wrappingType}
              <div class="summary-row">
                <span>Wrapping:</span>
                <span>{wrappingType}</span>
              </div>
            {/if}
            {#if cardStyle}
              <div class="summary-row">
                <span>Card:</span>
                <span>{cardStyle}</span>
              </div>
            {/if}
            {#if messageText}
              <div class="summary-row">
                <span>Message:</span>
                <span class="message-preview">{messageText.substring(0, 30)}{messageText.length > 30 ? '...' : ''}</span>
              </div>
            {/if}
            <div class="summary-row total">
              <span>Total Price:</span>
              <span class="final-price">{finalPrice.toLocaleString()} IQD</span>
            </div>
          </div>

          {#if errorMessage}
            <div class="error-message">{errorMessage}</div>
          {/if}

          <button
            class="pay-btn"
            on:click={handlePayment}
            disabled={!canPay}
          >
            {loadingPay ? 'Processing...' : 'Pay with SuperQi'}
          </button>
        </div>
      </div>
    </div>
  {/if}
</div>

<style>
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  }

  .app {
    min-height: 100vh;
    background: linear-gradient(135deg, #ffffff 0%, #f3f4f6 100%);
    padding-bottom: 40px;
  }

  .navbar {
    background: rgba(255, 255, 255, 0.55);
    backdrop-filter: blur(14px);
    padding: 16px 20px;
    text-align: center;
    position: sticky;
    top: 0;
    z-index: 100;
    border-bottom: 1px solid rgba(255, 255, 255, 0.3);
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
  }

  .navbar h1 {
    color: #2dd4bf;
    font-size: 24px;
    font-weight: 700;
    margin-bottom: 4px;
  }

  .user-info {
    font-size: 12px;
    color: #6b7280;
  }

  .auth-section {
    padding: 40px 20px;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 60vh;
  }

  .glass-card {
    background: rgba(255, 255, 255, 0.55);
    backdrop-filter: blur(14px);
    border-radius: 20px;
    padding: 24px;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.3);
  }

  .auth-card {
    max-width: 400px;
    width: 100%;
    text-align: center;
  }

  .auth-card h2 {
    color: #2dd4bf;
    margin-bottom: 12px;
    font-size: 22px;
  }

  .auth-card p {
    color: #6b7280;
    margin-bottom: 24px;
    font-size: 14px;
  }

  .auth-btn {
    width: 100%;
    padding: 14px 24px;
    background: linear-gradient(135deg, #2dd4bf 0%, #14b8a6 100%);
    color: white;
    border: none;
    border-radius: 12px;
    font-size: 16px;
    font-weight: 600;
    cursor: pointer;
    transition: transform 0.2s, box-shadow 0.2s;
  }

  .auth-btn:hover:not(:disabled) {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(45, 212, 191, 0.4);
  }

  .auth-btn:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  .category-bar {
    padding: 16px 0;
    background: rgba(255, 255, 255, 0.55);
    backdrop-filter: blur(14px);
    border-bottom: 1px solid rgba(255, 255, 255, 0.3);
    margin-bottom: 20px;
  }

  .category-scroll {
    display: flex;
    gap: 12px;
    padding: 0 20px;
    overflow-x: auto;
    scrollbar-width: none;
    -ms-overflow-style: none;
  }

  .category-scroll::-webkit-scrollbar {
    display: none;
  }

  .category-btn {
    padding: 10px 20px;
    background: rgba(255, 255, 255, 0.7);
    border: 1px solid rgba(255, 255, 255, 0.4);
    border-radius: 20px;
    color: #6b7280;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    white-space: nowrap;
    transition: all 0.2s;
  }

  .category-btn.active {
    background: linear-gradient(135deg, #2dd4bf 0%, #14b8a6 100%);
    color: white;
    border-color: #2dd4bf;
  }

  .category-btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  }

  .products-section {
    padding: 0 20px;
  }

  .section-title {
    color: #1f2937;
    font-size: 20px;
    margin-bottom: 16px;
    font-weight: 600;
  }

  .products-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 16px;
  }

  .product-card {
    text-align: center;
    padding: 16px;
    transition: transform 0.2s;
  }

  .product-card:hover {
    transform: translateY(-4px);
  }

  .product-card img {
    width: 100%;
    height: 140px;
    object-fit: contain;
    background: white;
    border-radius: 12px;
    margin-bottom: 12px;
  }

  .product-card h3 {
    color: #1f2937;
    font-size: 16px;
    margin-bottom: 8px;
    font-weight: 600;
  }

  .price {
    color: #2dd4bf;
    font-size: 18px;
    font-weight: 700;
    margin-bottom: 12px;
  }

  .customize-btn {
    width: 100%;
    padding: 10px;
    background: linear-gradient(135deg, #f9a8d4 0%, #f472b6 100%);
    color: white;
    border: none;
    border-radius: 10px;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    transition: transform 0.2s;
  }

  .customize-btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(249, 168, 212, 0.4);
  }

  .customize-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    backdrop-filter: blur(4px);
    z-index: 1000;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 20px;
    overflow-y: auto;
  }

  .customize-container {
    width: 100%;
    max-width: 500px;
    max-height: 90vh;
    overflow-y: auto;
  }

  .customize-card {
    position: relative;
    margin: 0;
  }

  .close-btn {
    position: absolute;
    top: 16px;
    right: 16px;
    width: 32px;
    height: 32px;
    background: rgba(255, 255, 255, 0.8);
    border: none;
    border-radius: 50%;
    font-size: 24px;
    color: #6b7280;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s;
  }

  .close-btn:hover {
    background: rgba(255, 255, 255, 1);
    transform: rotate(90deg);
  }

  .customize-card h2 {
    color: #2dd4bf;
    font-size: 22px;
    margin-bottom: 20px;
    text-align: center;
  }

  .product-preview {
    text-align: center;
    margin-bottom: 24px;
    padding-bottom: 20px;
    border-bottom: 1px solid rgba(0, 0, 0, 0.1);
  }

  .product-preview img {
    width: 100%;
    max-width: 200px;
    height: 200px;
    object-fit: contain;
    background: white;
    border-radius: 12px;
    margin-bottom: 12px;
  }

  .product-preview h3 {
    color: #1f2937;
    font-size: 18px;
    margin-bottom: 8px;
  }

  .base-price {
    color: #6b7280;
    font-size: 14px;
  }

  .customization-section {
    margin-bottom: 24px;
  }

  .customization-item {
    margin-bottom: 20px;
  }

  .customization-item label {
    display: block;
    color: #1f2937;
    font-size: 14px;
    font-weight: 600;
    margin-bottom: 8px;
  }

  .quantity-control {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 16px;
    background: rgba(255, 255, 255, 0.7);
    padding: 12px;
    border-radius: 12px;
  }

  .qty-btn {
    width: 36px;
    height: 36px;
    background: linear-gradient(135deg, #2dd4bf 0%, #14b8a6 100%);
    color: white;
    border: none;
    border-radius: 8px;
    font-size: 20px;
    font-weight: 600;
    cursor: pointer;
    transition: transform 0.2s;
  }

  .qty-btn:hover {
    transform: scale(1.1);
  }

  .qty-value {
    font-size: 18px;
    font-weight: 600;
    color: #1f2937;
    min-width: 30px;
    text-align: center;
  }

  .option-buttons {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
  }

  .option-btn {
    flex: 1;
    min-width: 80px;
    padding: 10px 16px;
    background: rgba(255, 255, 255, 0.7);
    border: 2px solid rgba(255, 255, 255, 0.4);
    border-radius: 10px;
    color: #6b7280;
    font-size: 13px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s;
  }

  .option-btn.selected {
    background: linear-gradient(135deg, #fdba74 0%, #fb923c 100%);
    color: white;
    border-color: #fdba74;
  }

  .option-btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  }

  .message-input {
    width: 100%;
    padding: 12px;
    background: rgba(255, 255, 255, 0.7);
    border: 2px solid rgba(255, 255, 255, 0.4);
    border-radius: 10px;
    font-size: 14px;
    font-family: inherit;
    color: #1f2937;
    resize: vertical;
    transition: border-color 0.2s;
  }

  .message-input:focus {
    outline: none;
    border-color: #2dd4bf;
    background: rgba(255, 255, 255, 0.9);
  }

  .order-summary {
    background: rgba(255, 255, 255, 0.7);
    padding: 16px;
    border-radius: 12px;
    margin-bottom: 20px;
  }

  .order-summary h3 {
    color: #1f2937;
    font-size: 16px;
    margin-bottom: 12px;
    font-weight: 600;
  }

  .summary-row {
    display: flex;
    justify-content: space-between;
    padding: 8px 0;
    font-size: 14px;
    color: #6b7280;
    border-bottom: 1px solid rgba(0, 0, 0, 0.05);
  }

  .summary-row:last-child {
    border-bottom: none;
  }

  .summary-row.total {
    margin-top: 8px;
    padding-top: 12px;
    border-top: 2px solid rgba(45, 212, 191, 0.3);
    font-weight: 600;
    color: #1f2937;
  }

  .final-price {
    color: #2dd4bf;
    font-size: 18px;
    font-weight: 700;
  }

  .message-preview {
    max-width: 200px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .pay-btn {
    width: 100%;
    padding: 16px;
    background: linear-gradient(135deg, #2dd4bf 0%, #14b8a6 100%);
    color: white;
    border: none;
    border-radius: 12px;
    font-size: 18px;
    font-weight: 700;
    cursor: pointer;
    transition: transform 0.2s, box-shadow 0.2s;
  }

  .pay-btn:hover:not(:disabled) {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(45, 212, 191, 0.4);
  }

  .pay-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .error-message {
    background: rgba(239, 68, 68, 0.1);
    color: #dc2626;
    padding: 12px;
    border-radius: 8px;
    font-size: 13px;
    margin-bottom: 16px;
    text-align: center;
    border: 1px solid rgba(239, 68, 68, 0.2);
  }
</style>
