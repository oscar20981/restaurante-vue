<template>
  <div id="app">

    <transition name="fade">
      <div v-if="loading" class="spinner-overlay">
        <div class="spinner-ring"></div>
        <p class="spinner-text">{{ spinnerMessage }}</p>
      </div>
    </transition>

    <transition-group name="toast" tag="div" class="toast-container">
      <div v-for="n in notifications" :key="n.id" :class="['toast', `toast-${n.type}`]">
        <span class="toast-dot"></span>{{ n.text }}
      </div>
    </transition-group>

    <!-- HEADER -->
    <header class="header">
      <div class="header-glow"></div>
      <div class="header-inner">
        <div class="brand">
          <div class="brand-icon">🔥</div>
          <div class="brand-text">
            <span class="brand-name">AMERICAN FOOD</span>
            <span class="brand-tagline">San Gil · Always Fresh</span>
          </div>
        </div>
        <nav class="header-actions">
          <button class="admin-toggle" @click="showAdmin = true" title="Admin Panel">
            ⚙️
          </button>
          <button class="cart-btn" @click="showCart = true">
            <span class="cart-icon">🛒</span>
            <span class="cart-label">Mi Pedido</span>
            <span v-if="cartCount() > 0" class="cart-badge">{{ cartCount() }}</span>
          </button>
        </nav>
      </div>
    </header>

    <!-- HERO -->
    <section class="hero">
      <div class="hero-mesh"></div>
      <div class="hero-dots"></div>
      <div class="hero-content">
        <p class="hero-eyebrow">🌶️ San Gil, Santander</p>
        <h1 class="hero-title">Comida que<br><em>enamora</em></h1>
        <p class="hero-sub">Domicilio gratis · Lun–Dom 10am–11pm · ☎ 320 5998 4265</p>
        <div class="hero-badges">
          <span class="hero-badge">⚡ Entrega rápida</span>
          <span class="hero-badge">🎁 Domicilio GRATIS</span>
          <span class="hero-badge">⭐ +500 pedidos</span>
        </div>
      </div>
      <div class="hero-food-strip">
        <span v-for="e in ['🍔','🌭','🍕','🌮','🍗','🍟','🥤','🍨']" :key="e" class="food-float">{{ e }}</span>
      </div>
    </section>

    <!-- SEARCH + CATS -->
    <div class="control-bar">
      <div class="control-bar-inner">
        <div class="search-wrap">
          <span class="search-icon-fixed">🔍</span>
          <input v-model="search" type="text" placeholder="Buscar producto..." class="search-input" />
        </div>
        <div class="cat-scroll">
          <button
            v-for="cat in categories" :key="cat.id"
            :class="['cat-pill', { active: selectedCat === cat.id }]"
            @click="selectedCat = cat.id"
          >{{ cat.icon }} {{ cat.name }}</button>
        </div>
      </div>
    </div>

    <!-- PRODUCTS -->
    <main class="products-main">
      <div v-if="getGroupedProducts().length === 0" class="no-results">
        <p class="no-results-emoji">😕</p>
        <p>No encontramos "<strong>{{ search }}</strong>"</p>
      </div>

      <section v-for="group in getGroupedProducts()" :key="group.catId" class="product-section">
        <div class="section-heading">
          <span class="section-emoji">{{ group.catIcon }}</span>
          <h2 class="section-name">{{ group.catName }}</h2>
          <span class="section-line"></span>
          <span class="section-count">{{ group.items.length }}</span>
        </div>

        <div class="product-grid">
          <div
            v-for="(p, idx) in group.items" :key="p.id"
            class="product-card"
            :style="{ animationDelay: (idx % 6) * 0.07 + 's' }"
            @click="openDetail(p)"
          >
            <div class="card-img">
              <img
                :src="p.image || getPlaceholderImg(p.emoji, p.name)"
                :alt="p.name"
                class="card-photo"
                @error="handleImgError($event, p)"
              />
              <span v-if="p.popular" class="hot-badge">🔥 TOP</span>
            </div>
            <div class="card-body">
              <h3 class="card-name">{{ p.name }}</h3>
              <p class="card-desc">{{ p.desc }}</p>
              <div class="card-foot" @click.stop>
                <span class="card-price">${{ fmt(p.price) }}</span>
                <div v-if="getQty(p.id) > 0" class="qty-row">
                  <button class="qty-btn" @click="dec(p.id)">−</button>
                  <span class="qty-val">{{ getQty(p.id) }}</span>
                  <button class="qty-btn" @click="inc(p)">+</button>
                </div>
                <button v-else class="add-btn" @click="inc(p)">+ Agregar</button>
              </div>
            </div>
          </div>
        </div>
      </section>
    </main>

    <!-- DETAIL MODAL — REDISEÑADO PARA LEGIBILIDAD -->
    <transition name="modal">
      <div v-if="detailProduct" class="overlay" @click.self="detailProduct = null">
        <div class="detail-card">
          <button class="x-btn detail-close" @click="detailProduct = null">✕</button>
          <div class="detail-img-wrap">
            <img
              :src="detailProduct.image || getPlaceholderImg(detailProduct.emoji, detailProduct.name)"
              :alt="detailProduct.name"
              class="detail-photo"
              @error="handleImgError($event, detailProduct)"
            />
            <div class="detail-img-overlay"></div>
            <span v-if="detailProduct.popular" class="detail-popular-badge">🔥 Más Pedido</span>
          </div>
          <div class="detail-body">
            <div class="detail-category-tag">{{ getCatName(detailProduct.cat) }}</div>
            <h2 class="detail-name">{{ detailProduct.name }}</h2>
            <p class="detail-desc">{{ detailProduct.fullDesc }}</p>

            <div class="detail-highlights" v-if="detailProduct.highlights && detailProduct.highlights.length">
              <div v-for="h in detailProduct.highlights" :key="h" class="detail-highlight-item">
                <span class="highlight-dot">●</span>{{ h }}
              </div>
            </div>

            <div class="detail-foot">
              <div class="detail-price-block">
                <span class="detail-price-label">Precio</span>
                <span class="detail-price">${{ fmt(detailProduct.price) }}</span>
              </div>
              <div v-if="getQty(detailProduct.id) > 0" class="qty-row lg">
                <button class="qty-btn lg" @click="dec(detailProduct.id)">−</button>
                <span class="qty-val lg">{{ getQty(detailProduct.id) }}</span>
                <button class="qty-btn lg" @click="inc(detailProduct)">+</button>
              </div>
              <button v-else class="cta-btn" @click="inc(detailProduct); detailProduct = null">
                🛒 Agregar al pedido
              </button>
            </div>
          </div>
        </div>
      </div>
    </transition>

    <!-- CART SIDEBAR -->
    <transition name="cart-overlay">
      <div v-if="showCart" class="overlay-bg" @click="showCart = false"></div>
    </transition>
    <transition name="slide-right">
      <div v-if="showCart" class="cart-sidebar">
        <div class="sidebar-head">
          <h2>Mi Pedido 🛒</h2>
          <button class="x-btn" @click="showCart = false">✕</button>
        </div>
        <div class="sidebar-body">
          <div v-if="cart.length === 0" class="empty-cart">
            <p class="empty-emoji">🍽️</p>
            <p class="empty-msg">Tu carrito está vacío</p>
            <p class="empty-sub">¡Agrega algo delicioso!</p>
          </div>
          <div v-else>
            <div v-for="item in cart" :key="item.id" class="cart-item">
              <div class="ci-img-wrap">
                <img
                  :src="item.image || getPlaceholderImg(item.emoji, item.name)"
                  :alt="item.name"
                  class="ci-img"
                  @error="$event.target.src = getEmojiImg(item.emoji)"
                />
              </div>
              <div class="ci-info">
                <p class="ci-name">{{ item.name }}</p>
                <div class="ci-row">
                  <div class="qty-row sm">
                    <button class="qty-btn" @click="dec(item.id)">−</button>
                    <span class="qty-val">{{ item.qty }}</span>
                    <button class="qty-btn" @click="inc(item)">+</button>
                  </div>
                  <span class="ci-total">${{ fmt(item.price * item.qty) }}</span>
                </div>
              </div>
              <button class="del-btn" @click="removeItem(item.id)">🗑</button>
            </div>
          </div>
        </div>
        <div v-if="cart.length > 0" class="sidebar-foot">
          <div class="summary-line">
            <span>Subtotal ({{ cartCount() }} items)</span>
            <span>${{ fmt(cartTotal()) }}</span>
          </div>
          <div class="summary-line">
            <span>Domicilio</span>
            <span class="free">GRATIS 🎁</span>
          </div>
          <div class="summary-line total">
            <span>TOTAL</span>
            <span>${{ fmt(cartTotal()) }}</span>
          </div>
          <button class="checkout-btn" @click="goCheckout">Confirmar pedido →</button>
        </div>
      </div>
    </transition>

    <!-- CHECKOUT MODAL -->
    <transition name="modal">
      <div v-if="showCheckout" class="overlay" @click.self="showCheckout = false">
        <div class="checkout-modal">

          <!-- PASO 1 -->
          <div v-if="step === 1">
            <div class="modal-head">
              <h2>📋 Tu información</h2>
              <button class="x-btn dark" @click="showCheckout = false">✕</button>
            </div>
            <div class="modal-body">
              <div class="form-row">
                <div class="form-group">
                  <label>Nombre *</label>
                  <input v-model.trim="form.nombre" :class="{ err: errors.nombre }" placeholder="Tu nombre" @input="clearErr('nombre')" />
                  <span v-if="errors.nombre" class="err-msg">{{ errors.nombre }}</span>
                </div>
                <div class="form-group">
                  <label>Apellido *</label>
                  <input v-model.trim="form.apellido" :class="{ err: errors.apellido }" placeholder="Tu apellido" @input="clearErr('apellido')" />
                  <span v-if="errors.apellido" class="err-msg">{{ errors.apellido }}</span>
                </div>
              </div>
              <div class="form-group">
                <label>WhatsApp *</label>
                <input v-model.trim="form.tel" :class="{ err: errors.tel }" placeholder="3001234567" maxlength="10" @input="clearErr('tel')" />
                <span v-if="errors.tel" class="err-msg">{{ errors.tel }}</span>
              </div>
              <div class="form-group">
                <label>Tipo de entrega</label>
                <select v-model="form.tipo" class="form-select">
                  <option value="domicilio">🛵 A domicilio</option>
                  <option value="local">🏪 Recoger en local</option>
                </select>
              </div>
              <div v-if="form.tipo === 'domicilio'" class="form-group">
                <label>Dirección *</label>
                <input v-model.trim="form.dir" :class="{ err: errors.dir }" placeholder="Calle, barrio..." @input="clearErr('dir')" />
                <span v-if="errors.dir" class="err-msg">{{ errors.dir }}</span>
              </div>
              <div class="form-group">
                <label>Método de pago *</label>
                <select v-model="form.pago" :class="{ err: errors.pago }" class="form-select" @change="clearErr('pago')">
                  <option value="">Seleccionar...</option>
                  <option value="efectivo">💵 Efectivo</option>
                  <option value="nequi">📱 Nequi</option>
                  <option value="daviplata">💙 Daviplata</option>
                  <option value="tarjeta">💳 Tarjeta</option>
                </select>
                <span v-if="errors.pago" class="err-msg">{{ errors.pago }}</span>
              </div>
              <div v-if="form.pago === 'efectivo'" class="form-group">
                <label>¿Con cuánto paga?</label>
                <input v-model.number="form.cambio" type="number" placeholder="Ej: 50000" />
              </div>
              <div class="form-group">
                <label>Notas (opcional)</label>
                <textarea v-model="form.nota" rows="2" placeholder="Sin cebolla, extra salsa..."></textarea>
              </div>
            </div>
            <div class="modal-foot">
              <button class="btn-sec" @click="showCheckout = false">Cancelar</button>
              <button class="btn-pri" @click="validateForm">Ver factura →</button>
            </div>
          </div>

          <!-- PASO 2: FACTURA REDISEÑADA -->
          <div v-if="step === 2">
            <div class="modal-head invoice-head">
              <h2>🧾 Factura del Pedido</h2>
              <button class="x-btn dark" @click="showCheckout = false">✕</button>
            </div>
            <div class="modal-body invoice-body">
              <div id="invoice-print" class="invoice">

                <!-- Header factura -->
                <div class="inv-header">
                  <div class="inv-logo-section">
                    <span class="inv-fire-icon">🔥</span>
                    <div class="inv-brand-info">
                      <div class="inv-brand-name">AMERICAN FOOD</div>
                      <div class="inv-brand-sub">San Gil, Santander · NIT: 900.123.456-7</div>
                    </div>
                  </div>
                  <div class="inv-meta">
                    <div class="inv-meta-item">
                      <span class="inv-meta-label">Factura N°</span>
                      <span class="inv-meta-value inv-num">{{ invoiceNum }}</span>
                    </div>
                    <div class="inv-meta-item">
                      <span class="inv-meta-label">Fecha</span>
                      <span class="inv-meta-value">{{ invoiceDate }}</span>
                    </div>
                  </div>
                </div>

                <!-- Datos cliente -->
                <div class="inv-section">
                  <div class="inv-section-title">
                    <span class="inv-section-icon">👤</span>
                    DATOS DEL CLIENTE
                  </div>
                  <div class="inv-client-grid">
                    <div class="inv-field">
                      <span class="inv-field-label">Cliente</span>
                      <span class="inv-field-value">{{ form.nombre }} {{ form.apellido }}</span>
                    </div>
                    <div class="inv-field">
                      <span class="inv-field-label">Teléfono</span>
                      <span class="inv-field-value">{{ form.tel }}</span>
                    </div>
                    <div class="inv-field">
                      <span class="inv-field-label">Entrega</span>
                      <span class="inv-field-value">{{ form.tipo === 'domicilio' ? '🛵 A domicilio' : '🏪 Recoger en local' }}</span>
                    </div>
                    <div class="inv-field">
                      <span class="inv-field-label">Pago</span>
                      <span class="inv-field-value">{{ pagoLabel() }}</span>
                    </div>
                    <div v-if="form.dir" class="inv-field inv-field-full">
                      <span class="inv-field-label">📍 Dirección</span>
                      <span class="inv-field-value">{{ form.dir }}</span>
                    </div>
                  </div>
                </div>

                <!-- Productos -->
                <div class="inv-section">
                  <div class="inv-section-title">
                    <span class="inv-section-icon">🛒</span>
                    DETALLE DEL PEDIDO
                  </div>
                  <div class="inv-products-list">
                    <div class="inv-product-header">
                      <span class="iph-name">Producto</span>
                      <span class="iph-qty">Cant.</span>
                      <span class="iph-unit">P. Unit.</span>
                      <span class="iph-total">Total</span>
                    </div>
                    <div
                      v-for="(item, i) in cart" :key="item.id"
                      :class="['inv-product-row', { 'inv-row-alt': i % 2 === 1 }]"
                    >
                      <span class="ipr-name">
                        <span class="ipr-emoji">{{ item.emoji }}</span>
                        {{ item.name }}
                      </span>
                      <span class="ipr-qty">
                        <span class="qty-badge">{{ item.qty }}</span>
                      </span>
                      <span class="ipr-unit">${{ fmt(item.price) }}</span>
                      <span class="ipr-total">${{ fmt(item.price * item.qty) }}</span>
                    </div>
                  </div>
                </div>

                <!-- Totales -->
                <div class="inv-totals-section">
                  <div class="inv-total-row">
                    <span class="itr-label">Subtotal</span>
                    <span class="itr-value">${{ fmt(cartTotal()) }}</span>
                  </div>
                  <div class="inv-total-row inv-row-green">
                    <span class="itr-label">🎁 Domicilio</span>
                    <span class="itr-value">GRATIS</span>
                  </div>
                  <div class="inv-grand-total">
                    <span class="igt-label">💰 TOTAL A PAGAR</span>
                    <span class="igt-amount">${{ fmt(cartTotal()) }}</span>
                  </div>
                  <div v-if="form.pago === 'efectivo' && form.cambio" class="inv-total-row inv-row-cambio">
                    <span class="itr-label">💵 Cambio estimado</span>
                    <span class="itr-value">${{ fmt(Math.max(0, form.cambio - cartTotal())) }}</span>
                  </div>
                </div>

                <div v-if="form.nota" class="inv-notes">
                  <span class="inv-notes-icon">📝</span>
                  <div>
                    <strong>Notas del pedido:</strong>
                    <span>{{ form.nota }}</span>
                  </div>
                </div>

                <div class="inv-footer-bar">
                  <span>¡GRACIAS POR TU PEDIDO!</span>
                  <span class="inv-footer-phone">☎ 320 5998 4265</span>
                </div>
              </div>
            </div>

            <div class="modal-foot">
              <button class="btn-sec" @click="step = 1">← Editar</button>
              <button class="btn-pdf" @click="downloadPDF">⬇ PDF</button>
              <button class="btn-pri" @click="confirmOrder">🚀 ¡Hacer pedido!</button>
            </div>
          </div>

          <!-- PASO 3: ÉXITO -->
          <div v-if="step === 3" class="success-box">
            <div class="success-icon">🎉</div>
            <h2 class="success-title">¡Pedido confirmado!</h2>
            <p class="success-sub">Tu pedido está siendo preparado con amor</p>
            <div class="order-chip">ORDEN {{ invoiceNum }}</div>
            <p class="success-time">{{ form.tipo === 'domicilio' ? '🛵 Entrega estimada: 25–40 min' : '🏪 Listo en: 15–20 min' }}</p>
            <button class="checkout-btn" @click="resetAll">Volver al menú</button>
          </div>
        </div>
      </div>
    </transition>

    <!-- ADMIN PANEL — COMPLETAMENTE REDISEÑADO -->
    <transition name="modal">
      <div v-if="showAdmin" class="overlay" @click.self="showAdmin = false">
        <div class="admin-modal">

          <!-- Cabecera clara -->
          <div class="admin-header">
            <div class="admin-header-left">
              <div class="admin-header-icon">⚙️</div>
              <div>
                <h2 class="admin-header-title">Agregar Producto</h2>
                <p class="admin-header-sub">Completa los campos para publicar en el menú</p>
              </div>
            </div>
            <button class="admin-close-btn" @click="showAdmin = false">✕</button>
          </div>

          <div class="admin-body">

            <!-- Columna izquierda: formulario -->
            <div class="admin-form-col">

              <div class="admin-fieldset">
                <div class="admin-fieldset-title">📝 Información básica</div>

                <div class="admin-field">
                  <label class="admin-label">Nombre del producto <span class="req">*</span></label>
                  <input v-model.trim="newProd.name" class="admin-input" placeholder="Ej: Burger Callejera" />
                </div>

                <div class="admin-field">
                  <label class="admin-label">Descripción corta <span class="req">*</span></label>
                  <input v-model.trim="newProd.desc" class="admin-input" placeholder="Ej: Carne 160g, queso, tocineta" />
                </div>

                <div class="admin-field">
                  <label class="admin-label">Descripción completa</label>
                  <textarea v-model="newProd.fullDesc" class="admin-textarea" rows="3" placeholder="Describe los ingredientes, el sabor, el acompañamiento..."></textarea>
                </div>

                <div class="admin-field">
                  <label class="admin-label">URL de imagen</label>
                  <input v-model.trim="newProd.image" class="admin-input" placeholder="https://... (jpg, png, webp)" />
                  <span class="admin-field-hint">Pega un link de imagen del producto. Si no tienes, usaremos el emoji.</span>
                </div>
              </div>

              <div class="admin-fieldset">
                <div class="admin-fieldset-title">💰 Precio y presentación</div>

                <div class="admin-two-col">
                  <div class="admin-field">
                    <label class="admin-label">Precio (COP) <span class="req">*</span></label>
                    <div class="admin-price-wrap">
                      <span class="admin-price-sym">$</span>
                      <input v-model.number="newProd.price" type="number" class="admin-input price-padded" placeholder="21900" />
                    </div>
                  </div>
                  <div class="admin-field">
                    <label class="admin-label">Emoji <span class="req">*</span></label>
                    <input v-model="newProd.emoji" class="admin-input emoji-centered" placeholder="🍔" maxlength="4" />
                  </div>
                </div>
              </div>

              <div class="admin-fieldset">
                <div class="admin-fieldset-title">🏷️ Clasificación</div>

                <div class="admin-two-col">
                  <div class="admin-field">
                    <label class="admin-label">Categoría <span class="req">*</span></label>
                    <select v-model="newProd.cat" class="admin-select">
                      <option v-for="c in categories.filter(c=>c.id!=='all')" :key="c.id" :value="c.id">
                        {{ c.icon }} {{ c.name }}
                      </option>
                    </select>
                  </div>
                  <div class="admin-field">
                    <label class="admin-label">¿Destacar producto?</label>
                    <button
                      type="button"
                      :class="['admin-toggle-btn', { 'admin-toggle-on': newProd.popular }]"
                      @click="newProd.popular = !newProd.popular"
                    >
                      <span class="admin-toggle-track">
                        <span class="admin-toggle-thumb"></span>
                      </span>
                      <span class="admin-toggle-text">{{ newProd.popular ? '🔥 Marcado como popular' : 'Producto normal' }}</span>
                    </button>
                  </div>
                </div>
              </div>

              <span v-if="adminError" class="admin-error">⚠️ {{ adminError }}</span>
            </div>

            <!-- Columna derecha: preview -->
            <div class="admin-preview-col">
              <div class="admin-preview-title">Vista previa de tarjeta</div>
              <div class="admin-preview-card" v-if="newProd.name">
                <div class="apc-img">
                  <img
                    v-if="newProd.image"
                    :src="newProd.image"
                    :alt="newProd.name"
                    class="apc-photo"
                    @error="$event.target.style.display='none'"
                  />
                  <span v-else class="apc-emoji">{{ newProd.emoji || '🍔' }}</span>
                  <span v-if="newProd.popular" class="apc-badge">🔥 TOP</span>
                </div>
                <div class="apc-body">
                  <h3 class="apc-name">{{ newProd.name || 'Nombre del producto' }}</h3>
                  <p class="apc-desc">{{ newProd.desc || 'Descripción del producto' }}</p>
                  <div class="apc-foot">
                    <span class="apc-price">${{ fmt(newProd.price || 0) }}</span>
                    <span class="apc-add-btn">+ Agregar</span>
                  </div>
                </div>
              </div>
              <div v-else class="admin-preview-empty">
                <span class="ape-icon">👀</span>
                <p>Comienza a llenar el formulario para ver la vista previa</p>
              </div>

              <!-- Info adicional -->
              <div class="admin-tips">
                <div class="admin-tips-title">💡 Consejos</div>
                <ul class="admin-tips-list">
                  <li>Usa una imagen cuadrada (1:1) para mejor apariencia</li>
                  <li>La descripción corta aparece en la tarjeta del menú</li>
                  <li>La descripción completa se muestra al abrir el producto</li>
                  <li>Marca como popular para que aparezca con la etiqueta 🔥 TOP</li>
                </ul>
              </div>
            </div>

          </div>

          <div class="admin-footer">
            <button class="btn-sec" @click="showAdmin = false">Cancelar</button>
            <button class="btn-pri admin-save-btn" @click="addProduct">
              ✅ Agregar al menú
            </button>
          </div>
        </div>
      </div>
    </transition>

    <!-- FOOTER -->
    <footer class="footer">
      <div class="footer-inner">
        <span class="footer-logo">🔥 American Food</span>
        <span class="footer-sep">·</span>
        <span>San Gil, Santander</span>
        <span class="footer-sep">·</span>
        <span>☎ 320 5998 4265</span>
        <span class="footer-sep">·</span>
        <span>Lun–Dom 10am–11pm</span>
      </div>
    </footer>

  </div>
</template>

<script setup>
import { ref, reactive } from 'vue'

const search = ref('')
const selectedCat = ref('all')
const showCart = ref(false)
const showCheckout = ref(false)
const showAdmin = ref(false)
const step = ref(1)
const loading = ref(false)
const spinnerMessage = ref('Procesando...')
const notifications = ref([])
const cart = ref([])
const detailProduct = ref(null)
const invoiceNum = ref('')
const invoiceDate = ref('')
const adminError = ref('')

const form = reactive({
  nombre: '', apellido: '', tel: '', tipo: 'domicilio',
  dir: '', pago: '', cambio: '', nota: ''
})
const errors = reactive({})

const newProd = reactive({
  name: '', desc: '', fullDesc: '', price: null,
  emoji: '', image: '', cat: 'hamburguesas', popular: false
})

const categories = [
  { id: 'all', name: 'Todo', icon: '🍽️' },
  { id: 'hamburguesas', name: 'Hamburguesas', icon: '🍔' },
  { id: 'perros', name: 'Perros', icon: '🌭' },
  { id: 'pizzas', name: 'Pizzas', icon: '🍕' },
  { id: 'tacos', name: 'Tacos', icon: '🌮' },
  { id: 'pollo', name: 'Pollo', icon: '🍗' },
  { id: 'acompa', name: 'Acompañamientos', icon: '🍟' },
  { id: 'bebidas', name: 'Bebidas', icon: '🥤' },
  { id: 'postres', name: 'Postres', icon: '🍨' },
]

// Imágenes reales de Unsplash (food photography libre)
const products = ref([
  {
    id: 1, name: 'Calle Bronx',
    desc: '160g carne res, mozzarella, cheddar',
    fullDesc: 'Nuestra hamburguesa más callejera: 160g de carne de res a la parrilla jugosa y dorada, queso mozzarella y cheddar fundidos, tocineta ahumada crujiente, cebolla caramelizada dorada, lechuga fresca y tomate. Salsas secretas de la casa. Acompañada de papas a la francesa doradas y crujientes.',
    highlights: ['Carne 160g a la parrilla', 'Doble queso fundido', 'Tocineta ahumada', 'Papas incluidas'],
    price: 21900, emoji: '🍔',
    image: 'https://images.unsplash.com/photo-1568901346375-23c9450c58cd?w=400&q=80',
    cat: 'hamburguesas', popular: true
  },
  {
    id: 2, name: 'Calle Picasso',
    desc: 'Pollo crispy, queso gouda, salsa BBQ',
    fullDesc: 'Pechuga de pollo crispy dorada y apanada artesanalmente, queso gouda fundido, tocineta, vegetales frescos, lechuga crujiente, tomate maduro y nuestra salsa BBQ especial hecha en casa. Acompañada con papas a la francesa crujientes.',
    highlights: ['Pollo crispy artesanal', 'Gouda fundido', 'BBQ casera', 'Papas incluidas'],
    price: 21900, emoji: '🍔',
    image: 'https://images.unsplash.com/photo-1586190848861-99aa4a171e90?w=400&q=80',
    cat: 'hamburguesas', popular: true
  },
  {
    id: 3, name: 'Calle Campesina',
    desc: 'Doble carne angus, queso cheddar doble',
    fullDesc: 'Doble carne angus de 200g total, con queso cheddar doble completamente derretido, tocino ahumado crujiente, lechuga romana, tomate fresco, cebolla morada y nuestra exclusiva salsa campesina. Acompañada con papas a la francesa.',
    highlights: ['Doble carne angus 200g', 'Cheddar doble', 'Salsa campesina exclusiva', 'Papas incluidas'],
    price: 23900, emoji: '🍔',
    image: 'https://images.unsplash.com/photo-1553979459-d2229ba7433b?w=400&q=80',
    cat: 'hamburguesas', popular: false
  },
  {
    id: 4, name: 'Calle Parlache',
    desc: 'Triple carne, onion ring, queso fundido',
    fullDesc: 'La reina del menú. Triple carne de 280g total, cheddar y mozzarella fundidos generosamente, onion ring crocante encima, tocino doble, jalapeños para el que le gusta el picante, vegetales frescos y las salsas secretas de Callejeros. Para los valientes.',
    highlights: ['Triple carne 280g', 'Doble queso fundido', 'Onion ring incluido', 'Jalapeños frescos'],
    price: 24900, emoji: '🍔',
    image: 'https://images.unsplash.com/photo-1594212699903-ec8a3eca50f5?w=400&q=80',
    cat: 'hamburguesas', popular: false
  },
  {
    id: 5, name: 'Calle Graffiti',
    desc: 'Carne, guacamole, pico de gallo, jalapeños',
    fullDesc: 'Con toque mexicano auténtico. Carne a la parrilla 140g con guacamole fresco preparado el mismo día, pico de gallo casero, jalapeños frescos, queso oaxaca fundido y crema agria. Acompañada con papas tipo cajún con especias.',
    highlights: ['Guacamole fresco del día', 'Pico de gallo casero', 'Queso oaxaca', 'Papas cajún incluidas'],
    price: 22900, emoji: '🍔',
    image: 'https://images.unsplash.com/photo-1609167830220-7164aa360951?w=400&q=80',
    cat: 'hamburguesas', popular: false
  },
  {
    id: 6, name: 'Chicken Callejero',
    desc: 'Pechuga apanada, coleslaw casero',
    fullDesc: 'Pechuga de pollo apanada estilo sureño americano, dorada y crujiente. Ensalada coleslaw hecha en casa con zanahoria y repollo, pepinillos encurtidos artesanales, queso cheddar y mayonesa de ajo roasted. Con papas a la francesa crujientes.',
    highlights: ['Estilo sureño americano', 'Coleslaw casero', 'Pepinillos artesanales', 'Mayonesa de ajo'],
    price: 19900, emoji: '🍔',
    image: 'https://images.unsplash.com/photo-1572802419224-296b0aeee0d9?w=400&q=80',
    cat: 'hamburguesas', popular: false
  },
  {
    id: 7, name: 'Perro Clásico',
    desc: 'Salchicha, papas chips, queso, salsas',
    fullDesc: 'La estrella del local desde el primer día. Salchicha jumbo a la plancha con papas chips crujientes, queso rallado generoso, piña troceada, y las clásicas salsas callejeras: rosada, mostaza, kétchup y mayonesa. El sabor de la calle en tu mano.',
    highlights: ['Salchicha jumbo', 'Papas chips crujientes', 'Queso rallado', '4 salsas incluidas'],
    price: 9900, emoji: '🌭',
    image: 'https://images.unsplash.com/photo-1612392166886-ee8475b03af2?w=400&q=80',
    cat: 'perros', popular: true
  },
  {
    id: 8, name: 'Perro Relleno',
    desc: 'Salchicha rellena mozarella, tocino',
    fullDesc: 'Salchicha premium rellena de queso mozarella que se derrite al morderla, envuelta en tocino crujiente ahumado, con papas chips por encima, salsas especiales de la casa y queso parmesano recién rallado. Una experiencia diferente.',
    highlights: ['Rellena de mozarella', 'Envuelta en tocino', 'Queso parmesano', 'Papas chips'],
    price: 12900, emoji: '🌭',
    image: '',
    cat: 'perros', popular: false
  },
  {
    id: 9, name: 'Mega Perro XXL',
    desc: 'Salchicha 25cm, maíz, pimentón, 5 salsas',
    fullDesc: 'El perro más grande del menú. Salchicha XXL de 25cm a la plancha con papas fritas largas, maíz tierno dulce, pimentón rojo en tiras, cebolla caramelizada dorada, queso fundido abundante y las 5 salsas especiales de la casa. Para compartir... o no.',
    highlights: ['Salchicha XXL 25cm', 'Maíz y pimentón', 'Cebolla caramelizada', '5 salsas de la casa'],
    price: 15900, emoji: '🌭',
    image: '',
    cat: 'perros', popular: false
  },
  {
    id: 10, name: 'Perro BBQ Smoky',
    desc: 'Salchicha ahumada, BBQ artesanal',
    fullDesc: 'Salchicha ahumada especial a la plancha hasta dorar perfectamente, bañada generosamente en nuestra salsa BBQ artesanal cocinada en casa. Cebolla caramelizada lentamente, queso gouda derretido y papas tipo rústico.',
    highlights: ['Salchicha ahumada especial', 'BBQ artesanal de la casa', 'Queso gouda', 'Papas rústicas'],
    price: 13900, emoji: '🌭',
    image: '',
    cat: 'perros', popular: false
  },
  {
    id: 11, name: 'Pizza Margherita',
    desc: 'Tomate, mozzarella fresca, albahaca',
    fullDesc: 'La clásica italiana hecha con amor. Base de tomate artesanal preparada a diario, mozzarella fresca importada en abundancia, albahaca fresca recogida ese día y aceite de oliva virgen extra. Horneada a alta temperatura para base crujiente y centro suave.',
    highlights: ['Mozzarella fresca importada', 'Albahaca fresca del día', 'Aceite de oliva virgen', 'Horneada a alta temperatura'],
    price: 22900, emoji: '🍕',
    image: 'https://images.unsplash.com/photo-1574071318508-1cdbab80d002?w=400&q=80',
    cat: 'pizzas', popular: false
  },
  {
    id: 12, name: 'Pizza Callejera',
    desc: 'Pepperoni, tocino, jalapeños, mozzarella',
    fullDesc: 'Nuestra pizza insignia y la favorita del barrio. Pepperoni importado premium, tocino ahumado crujiente, jalapeños frescos que pican justo, mozzarella derretida y dorada, todo sobre nuestra salsa de tomate picante casera. Imprescindible.',
    highlights: ['Pepperoni importado premium', 'Tocino ahumado', 'Jalapeños frescos', 'Salsa picante casera'],
    price: 26900, emoji: '🍕',
    image: 'https://images.unsplash.com/photo-1565299624946-b28f40a0ae38?w=400&q=80',
    cat: 'pizzas', popular: true
  },
  {
    id: 13, name: 'Taco Callejero',
    desc: 'Carne asada, guacamole, pico de gallo',
    fullDesc: 'Tortilla de maíz tostada en comal con carne asada marinada en especias mexicanas durante horas, guacamole fresco del día, pico de gallo casero con jitomate y cilantro, cebolla blanca y salsa verde taquera. Porción de 3 tacos generosos.',
    highlights: ['Carne marinada en especias', 'Guacamole fresco', 'Pico de gallo casero', 'Porción de 3 tacos'],
    price: 18900, emoji: '🌮',
    image: '',
    cat: 'tacos', popular: true
  },
  {
    id: 14, name: 'Taco Pollo BBQ',
    desc: 'Pollo BBQ, maíz, crema, queso',
    fullDesc: 'Pollo a la parrilla marinado y bañado en salsa BBQ artesanal de la casa, maíz tostado dulce y ahumado, crema agria fresca, queso fresco rallado y jalapeños opcionales en tortilla de maíz artesanal. Porción de 3 tacos.',
    highlights: ['Pollo BBQ artesanal', 'Maíz tostado ahumado', 'Queso fresco rallado', 'Porción de 3 tacos'],
    price: 17900, emoji: '🌮',
    image: '',
    cat: 'tacos', popular: false
  },
  {
    id: 15, name: 'Alitas BBQ',
    desc: '8 unidades, salsa BBQ artesanal',
    fullDesc: '8 alitas de pollo horneadas a la perfección y bañadas en generosa salsa BBQ artesanal hecha en casa. Crujientes por fuera, jugosas y tiernas por dentro. Acompañadas con salsa ranch cremosa y bastones de apio fresco para limpiar el paladar.',
    highlights: ['8 alitas de pollo', 'BBQ artesanal de la casa', 'Crujientes por fuera', 'Con ranch y apio'],
    price: 19900, emoji: '🍗',
    image: 'https://images.unsplash.com/photo-1527477396000-e27163b481c2?w=400&q=80',
    cat: 'pollo', popular: true
  },
  {
    id: 16, name: 'Pollo Crispy',
    desc: 'Pechuga apanada, papas, ensalada',
    fullDesc: 'Pechuga de pollo apanada a la perfección con mezcla de especias secretas, dorada en aceite hasta quedar completamente crujiente. Acompañada de papas a la francesa doradas y ensalada coleslaw casera con aderezo de mostaza y miel.',
    highlights: ['Apanado con especias secretas', 'Dorado y crujiente', 'Coleslaw casero', 'Papas incluidas'],
    price: 18900, emoji: '🍗',
    image: '',
    cat: 'pollo', popular: false
  },
  {
    id: 17, name: 'Papas Fritas',
    desc: 'Papas crujientes, sal marina',
    fullDesc: 'Papas cortadas a mano en casa todos los días, fritas en aceite caliente a la perfección hasta quedar doradas y crujientes por fuera, suaves por dentro. Sazonadas con sal marina en el momento. Acompañadas con la salsa de tu elección: kétchup, mayonesa, BBQ o rosada.',
    highlights: ['Cortadas a mano cada día', 'Fritas al momento', 'Sal marina premium', 'Salsa a elegir'],
    price: 6900, emoji: '🍟',
    image: 'https://images.unsplash.com/photo-1573080496219-bb080dd4f877?w=400&q=80',
    cat: 'acompa', popular: false
  },
  {
    id: 18, name: 'Papas Gourmet',
    desc: 'Papas, queso cheddar, tocineta, cebollín',
    fullDesc: 'Papas fritas crujientes bañadas generosamente en queso cheddar derretido caliente, tocineta crujiente picada por encima, cebollín fresco y jalapeños opcionales. El acompañamiento gourmet perfecto para llevar tu combo al siguiente nivel.',
    highlights: ['Queso cheddar fundido', 'Tocineta crujiente', 'Cebollín fresco', 'Jalapeños opcionales'],
    price: 10900, emoji: '🍟',
    image: 'https://images.unsplash.com/photo-1630431341973-02e1b662ec35?w=400&q=80',
    cat: 'acompa', popular: true
  },
  {
    id: 19, name: 'Onion Rings',
    desc: 'Aros de cebolla apanados, salsa dip',
    fullDesc: '8 aros de cebolla grandes, cortados gruesos, apanados con pan rallado especiado con hierbas y fritos hasta quedar perfectamente dorados y crujientes. Acompañados con nuestra salsa dip especial de la casa. El complemento perfecto.',
    highlights: ['8 aros grandes', 'Apanado con hierbas', 'Fritos al momento', 'Salsa dip de la casa'],
    price: 8900, emoji: '🍟',
    image: 'https://images.unsplash.com/photo-1639024471283-03518883512d?w=400&q=80',
    cat: 'acompa', popular: false
  },
  {
    id: 20, name: 'Gaseosa 400ml',
    desc: 'Coca-Cola, Pepsi, Sprite o Manzana',
    fullDesc: 'Gaseosa fría a tu elección servida con hielo: Coca-Cola clásica, Pepsi, Sprite o Manzana. Perfecta y refrescante para acompañar cualquier plato del menú. Siempre bien fría.',
    highlights: ['4 sabores disponibles', 'Servida con hielo', 'Siempre bien fría', '400ml'],
    price: 4000, emoji: '🥤',
    image: 'https://images.unsplash.com/photo-1625772299848-391b6a87d7b3?w=400&q=80',
    cat: 'bebidas', popular: false
  },
  {
    id: 21, name: 'Malteada',
    desc: 'Vainilla, chocolate o fresa',
    fullDesc: 'Malteada cremosa y espesa preparada al momento con helado artesanal de alta calidad. Elige entre el clásico de vainilla, el intenso chocolate o la dulce fresa. Tamaño grande de 500ml, cubierta de crema batida.',
    highlights: ['3 sabores artesanales', 'Preparada al momento', '500ml tamaño grande', 'Con crema batida'],
    price: 9900, emoji: '🥤',
    image: 'https://images.unsplash.com/photo-1568901839119-631418a3910d?w=400&q=80',
    cat: 'bebidas', popular: true
  },
  {
    id: 22, name: 'Agua Cristal',
    desc: 'Agua mineral natural 600ml',
    fullDesc: 'Agua mineral natural Cristal 600ml, sin gas, refrescante y natural. La opción más saludable y ligera del menú. Siempre fría y lista para acompañar tu pedido.',
    highlights: ['Agua natural sin gas', '600ml', 'Siempre fría', 'La opción saludable'],
    price: 2500, emoji: '💧',
    image: 'https://images.unsplash.com/photo-1548839140-29a749e1cf4d?w=400&q=80',
    cat: 'bebidas', popular: false
  },
  {
    id: 23, name: 'Helado Artesanal',
    desc: '2 bolas, sabores variados',
    fullDesc: 'Helado artesanal preparado con ingredientes naturales y sin conservantes artificiales. Elige 2 bolas generosas de tus sabores favoritos: vainilla clásica, chocolate intenso, arequipe colombiano o fresa natural. Servido en copa o cono.',
    highlights: ['Ingredientes 100% naturales', '2 bolas generosas', '4 sabores disponibles', 'Copa o cono a elegir'],
    price: 7900, emoji: '🍨',
    image: 'https://images.unsplash.com/photo-1563805042-7684c019e1cb?w=400&q=80',
    cat: 'postres', popular: false
  },
  {
    id: 24, name: 'Brownie Callejero',
    desc: 'Brownie con helado y chocolate',
    fullDesc: 'El postre perfecto para cerrar con broche de oro. Brownie de chocolate oscuro tibio y húmedo, acompañado de una bola generosa de helado de vainilla artesanal derritiéndose encima y salsa de chocolate artesanal. Un pecado delicioso.',
    highlights: ['Brownie de chocolate oscuro', 'Helado de vainilla artesanal', 'Salsa de chocolate casera', 'Servido tibio'],
    price: 10900, emoji: '🍫',
    image: 'https://images.unsplash.com/photo-1564355808539-22fda35bed7e?w=400&q=80',
    cat: 'postres', popular: true
  },
])

let nextId = 100

function fmt(n) { return Number(n).toLocaleString('es-CO') }
function cartCount() { return cart.value.reduce((s, i) => s + i.qty, 0) }
function cartTotal() { return cart.value.reduce((s, i) => s + i.price * i.qty, 0) }
function pagoLabel() {
  return { efectivo: '💵 Efectivo', nequi: '📱 Nequi', daviplata: '💙 Daviplata', tarjeta: '💳 Tarjeta' }[form.pago] || ''
}
function getCatName(catId) {
  const cat = categories.find(c => c.id === catId)
  return cat ? `${cat.icon} ${cat.name}` : ''
}
function getGroupedProducts() {
  const q = search.value.toLowerCase().trim()
  const cats = selectedCat.value === 'all'
    ? categories.filter(c => c.id !== 'all')
    : categories.filter(c => c.id === selectedCat.value)
  return cats.map(cat => {
    let items = products.value.filter(p => p.cat === cat.id)
    if (q) items = items.filter(p => p.name.toLowerCase().includes(q) || p.desc.toLowerCase().includes(q))
    return { catId: cat.id, catName: cat.name, catIcon: cat.icon, items }
  }).filter(g => g.items.length > 0)
}
function getQty(id) { return cart.value.find(i => i.id === id)?.qty ?? 0 }
function getPlaceholderImg(emoji, name) {
  // Fallback con color basado en el nombre
  return `https://via.placeholder.com/400x400/FFF0E0/C8271E?text=${encodeURIComponent(emoji)}`
}
function getEmojiImg(emoji) {
  return `https://via.placeholder.com/60x60/FFF0E0/C8271E?text=${encodeURIComponent(emoji)}`
}
function handleImgError(event, product) {
  event.target.src = getPlaceholderImg(product.emoji, product.name)
}
function inc(product) {
  const ex = cart.value.find(i => i.id === product.id)
  if (ex) { ex.qty++ } else {
    cart.value.push({
      id: product.id, name: product.name, emoji: product.emoji,
      image: product.image, price: product.price, qty: 1
    })
    notify(`${product.emoji} ${product.name} agregado`, 'success')
  }
}
function dec(id) {
  const item = cart.value.find(i => i.id === id)
  if (!item) return
  if (item.qty > 1) item.qty--; else removeItem(id)
}
function removeItem(id) {
  const idx = cart.value.findIndex(i => i.id === id)
  if (idx !== -1) { const n = cart.value[idx].name; cart.value.splice(idx, 1); notify(`${n} eliminado`, 'info') }
}
function openDetail(p) { detailProduct.value = p }
function goCheckout() {
  if (!cart.value.length) return
  showCart.value = false
  step.value = 1
  showCheckout.value = true
}
function clearErr(f) { delete errors[f] }
function validateForm() {
  Object.keys(errors).forEach(k => delete errors[k])
  let ok = true
  if (!form.nombre) { errors.nombre = 'Requerido'; ok = false }
  if (!form.apellido) { errors.apellido = 'Requerido'; ok = false }
  if (!form.tel) { errors.tel = 'Requerido'; ok = false }
  else if (!/^3\d{9}$/.test(form.tel)) { errors.tel = 'Número inválido (10 dígitos, empieza en 3)'; ok = false }
  if (form.tipo === 'domicilio' && !form.dir) { errors.dir = 'Dirección requerida'; ok = false }
  if (!form.pago) { errors.pago = 'Selecciona un método'; ok = false }
  if (!ok) { notify('Corrige los errores marcados', 'error'); return }
  invoiceNum.value = 'AF-' + Date.now().toString().slice(-5)
  invoiceDate.value = new Date().toLocaleDateString('es-CO', { day: '2-digit', month: 'short', year: 'numeric' })
  step.value = 2
}
function confirmOrder() {
  showCheckout.value = false
  loading.value = true
  spinnerMessage.value = 'Enviando pedido...'
  setTimeout(() => { spinnerMessage.value = 'Confirmando con la cocina...' }, 1200)
  setTimeout(() => {
    loading.value = false
    showCheckout.value = true
    step.value = 3
    notify('¡Pedido confirmado! 🎉', 'success')
  }, 2400)
}
function resetAll() {
  cart.value = []
  Object.assign(form, { nombre: '', apellido: '', tel: '', tipo: 'domicilio', dir: '', pago: '', cambio: '', nota: '' })
  step.value = 1
  showCheckout.value = false
  notify('¡Gracias! Vuelve pronto 🔥', 'success')
}

async function downloadPDF() {
  notify('Generando PDF...', 'info')
  const loadScript = src => new Promise((res, rej) => {
    if (document.querySelector(`script[src="${src}"]`)) return res()
    const s = document.createElement('script')
    s.src = src; s.onload = res; s.onerror = rej
    document.head.appendChild(s)
  })
  try {
    await loadScript('https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js')
    await loadScript('https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js')
    const el = document.getElementById('invoice-print')
    const canvas = await window.html2canvas(el, { scale: 2, backgroundColor: '#ffffff', useCORS: true })
    const imgData = canvas.toDataURL('image/png')
    const { jsPDF } = window.jspdf
    const pdf = new jsPDF({ unit: 'mm', format: 'a4', orientation: 'portrait' })
    const pageW = pdf.internal.pageSize.getWidth()
    const imgW = pageW - 20
    const imgH = (canvas.height * imgW) / canvas.width
    pdf.addImage(imgData, 'PNG', 10, 10, imgW, imgH)
    pdf.save(`Factura-${invoiceNum.value}.pdf`)
    notify('PDF descargado ✅', 'success')
  } catch (e) {
    notify('Error generando PDF', 'error')
    console.error(e)
  }
}

function addProduct() {
  adminError.value = ''
  if (!newProd.name.trim()) { adminError.value = 'El nombre es obligatorio'; return }
  if (!newProd.desc.trim()) { adminError.value = 'La descripción es obligatoria'; return }
  if (!newProd.price || newProd.price <= 0) { adminError.value = 'Ingresa un precio válido'; return }
  if (!newProd.emoji.trim()) { adminError.value = 'Agrega un emoji para el producto'; return }
  products.value.push({
    id: ++nextId,
    name: newProd.name.trim(),
    desc: newProd.desc.trim(),
    fullDesc: newProd.fullDesc.trim() || newProd.desc.trim(),
    price: newProd.price,
    emoji: newProd.emoji.trim(),
    image: newProd.image.trim() || null,
    highlights: [],
    cat: newProd.cat,
    popular: newProd.popular
  })
  Object.assign(newProd, { name: '', desc: '', fullDesc: '', price: null, emoji: '', image: '', cat: 'hamburguesas', popular: false })
  showAdmin.value = false
  notify('✅ Producto agregado al menú', 'success')
}

let notifId = 0
function notify(text, type = 'info') {
  const id = ++notifId
  notifications.value.push({ id, text, type })
  setTimeout(() => {
    const i = notifications.value.findIndex(n => n.id === id)
    if (i !== -1) notifications.value.splice(i, 1)
  }, 3200)
}
</script>

<style scoped>
/* ═══════ TOKENS ═══════ */
:root {
  --red: #C8271E; --red2: #A01F17; --red3: #FF4B3E;
  --amber: #F59E0B; --gold: #FFD166;
  --bg: #FDF6EE; --bg2: #FFF0E0; --card: #FFFFFF;
  --text: #1C1410; --muted: #7A6B60; --border: #E8DDD4;
  --radius: 16px; --radius-sm: 10px;
  --shadow: 0 2px 12px rgba(200,39,30,.1);
  --shadow-lg: 0 12px 40px rgba(0,0,0,.18);
}
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
html { font-family: 'Georgia', 'Palatino', serif; background: var(--bg); color: var(--text); scroll-behavior: smooth; }
#app { min-height: 100vh; display: flex; flex-direction: column; }

/* ═══════ HEADER ═══════ */
.header {
  position: relative;
  background: linear-gradient(135deg, #1C0A05 0%, #2D1008 50%, #1C0A05 100%);
  border-bottom: 4px solid var(--red);
  box-shadow: 0 4px 30px rgba(200,39,30,.4);
  overflow: hidden;
}
.header-glow {
  position: absolute; top: -60px; left: 50%; transform: translateX(-50%);
  width: 400px; height: 120px;
  background: radial-gradient(ellipse, rgba(200,39,30,.5) 0%, transparent 70%);
  pointer-events: none;
}
.header-inner {
  max-width: 1400px; margin: 0 auto; padding: 16px 28px;
  display: flex; align-items: center; justify-content: space-between; gap: 16px;
  position: relative; z-index: 1;
}
.brand { display: flex; align-items: center; gap: 14px; }
.brand-icon { font-size: 2.2rem; filter: drop-shadow(0 0 12px rgba(255,100,50,.6)); }
.brand-name { display: block; font-family: 'Georgia', serif; font-size: 1.5rem; font-weight: 900; letter-spacing: 3px; color: #FFFFFF; text-transform: uppercase; line-height: 1.1; text-shadow: 0 0 20px rgba(255,150,50,.5); }
.brand-tagline { display: block; font-size: .7rem; color: rgba(255,220,180,.7); letter-spacing: 2.5px; text-transform: uppercase; font-family: sans-serif; }
.header-actions { display: flex; align-items: center; gap: 10px; }
.admin-toggle {
  background: rgba(255,255,255,.08); border: 1.5px solid rgba(255,255,255,.2);
  border-radius: var(--radius-sm); width: 44px; height: 44px;
  cursor: pointer; font-size: 1.1rem;
  display: flex; align-items: center; justify-content: center;
  transition: all .2s; color: #fff;
}
.admin-toggle:hover { background: rgba(255,255,255,.16); border-color: var(--gold); }
.cart-btn {
  position: relative;
  background: linear-gradient(135deg, var(--red) 0%, var(--red2) 100%);
  color: #fff; border: 2px solid rgba(255,255,255,.25);
  border-radius: var(--radius-sm); padding: 11px 20px; cursor: pointer;
  font-family: sans-serif; font-size: .82rem; font-weight: 800;
  letter-spacing: 1.5px; text-transform: uppercase;
  display: flex; align-items: center; gap: 8px; transition: all .2s;
  box-shadow: 0 4px 16px rgba(200,39,30,.5);
}
.cart-btn:hover { transform: translateY(-2px); box-shadow: 0 8px 24px rgba(200,39,30,.6); }
.cart-badge {
  position: absolute; top: -10px; right: -10px;
  background: var(--gold); color: #1C0A05; width: 24px; height: 24px;
  border-radius: 50%; font-size: .75rem; font-weight: 900;
  display: flex; align-items: center; justify-content: center;
  animation: pulse .9s ease infinite; border: 2.5px solid #1C0A05; font-family: sans-serif;
}
@keyframes pulse { 0%,100%{transform:scale(1)} 50%{transform:scale(1.2)} }

/* ═══════ HERO ═══════ */
.hero {
  position: relative; overflow: hidden;
  background: linear-gradient(160deg, #1C0A05 0%, #3D1208 40%, #5C1E0A 70%, #3D1208 100%);
  padding: 64px 28px 28px;
}
.hero-mesh {
  position: absolute; inset: 0;
  background: radial-gradient(circle at 20% 50%, rgba(200,39,30,.35) 0%, transparent 50%),
    radial-gradient(circle at 80% 30%, rgba(245,158,11,.2) 0%, transparent 45%),
    radial-gradient(circle at 60% 80%, rgba(200,39,30,.2) 0%, transparent 40%);
  pointer-events: none;
}
.hero-dots {
  position: absolute; inset: 0; opacity: .07;
  background-image: radial-gradient(circle, rgba(255,220,150,.8) 1px, transparent 1px);
  background-size: 28px 28px; pointer-events: none;
}
.hero-content { position: relative; z-index: 1; max-width: 1400px; margin: 0 auto; }
.hero-eyebrow { font-size: .82rem; font-weight: 700; color: var(--gold); letter-spacing: 3px; text-transform: uppercase; margin-bottom: 12px; font-family: sans-serif; }
.hero-title { font-family: 'Georgia', serif; font-size: clamp(2.6rem, 6vw, 5rem); font-weight: 900; line-height: 1.05; color: #FFFFFF; margin-bottom: 16px; text-shadow: 0 4px 30px rgba(0,0,0,.5); }
.hero-title em { color: var(--red3); font-style: italic; text-shadow: 0 0 40px rgba(200,39,30,.6); }
.hero-sub { font-size: .92rem; color: rgba(255,220,180,.8); font-weight: 600; font-family: sans-serif; margin-bottom: 20px; }
.hero-badges { display: flex; flex-wrap: wrap; gap: 8px; margin-top: 4px; }
.hero-badge {
  background: rgba(255,255,255,.1); color: rgba(255,240,220,.9);
  border: 1px solid rgba(255,200,100,.3); border-radius: 999px;
  padding: 5px 14px; font-size: .75rem; font-weight: 700; letter-spacing: .5px;
  font-family: sans-serif; backdrop-filter: blur(4px);
}
.hero-food-strip { display: flex; gap: 22px; padding: 28px 0 6px; max-width: 1400px; margin: 0 auto; position: relative; z-index: 1; }
.food-float { font-size: 2.4rem; opacity: .5; filter: drop-shadow(0 4px 8px rgba(0,0,0,.4)); animation: floatY 3s ease-in-out infinite; }
.food-float:nth-child(even) { animation-delay: .5s; }
.food-float:nth-child(3n) { animation-delay: 1s; }
@keyframes floatY { 0%,100%{transform:translateY(0)} 50%{transform:translateY(-8px)} }

/* ═══════ CONTROL BAR ═══════ */
.control-bar { background: var(--card); border-bottom: 2px solid var(--border); box-shadow: 0 4px 20px rgba(0,0,0,.07); }
.control-bar-inner { max-width: 1400px; margin: 0 auto; padding: 14px 28px; display: flex; flex-direction: column; gap: 12px; }
.search-wrap { position: relative; }
.search-icon-fixed { position: absolute; left: 16px; top: 50%; transform: translateY(-50%); font-size: 1rem; pointer-events: none; }
.search-input {
  width: 100%; border: 2px solid var(--border); border-radius: var(--radius-sm);
  background: var(--bg); padding: 11px 16px 11px 46px;
  font-size: .95rem; font-weight: 600; color: var(--text); outline: none; transition: all .2s; font-family: sans-serif;
}
.search-input:focus { border-color: var(--red); background: #fff; box-shadow: 0 0 0 4px rgba(200,39,30,.08); }
.search-input::placeholder { color: #C4B8B0; }
.cat-scroll { display: flex; gap: 8px; overflow-x: auto; scrollbar-width: none; padding-bottom: 2px; }
.cat-scroll::-webkit-scrollbar { display: none; }
.cat-pill {
  padding: 7px 16px; white-space: nowrap; border: 2px solid var(--border);
  border-radius: 999px; background: var(--bg); color: var(--muted);
  font-size: .8rem; font-weight: 700; letter-spacing: .5px; cursor: pointer; transition: all .18s; font-family: sans-serif;
}
.cat-pill:hover { border-color: var(--red); color: var(--red); background: #fff; }
.cat-pill.active { background: var(--red); border-color: var(--red); color: #fff; box-shadow: 0 3px 12px rgba(200,39,30,.35); }

/* ═══════ PRODUCTS ═══════ */
.products-main { max-width: 1400px; margin: 0 auto; padding: 40px 28px 72px; flex: 1; }
.no-results { text-align: center; padding: 90px 20px; }
.no-results-emoji { font-size: 3rem; margin-bottom: 14px; }
.no-results p { color: var(--muted); font-size: 1rem; font-family: sans-serif; }
.product-section { margin-bottom: 56px; }
.section-heading { display: flex; align-items: center; gap: 12px; margin-bottom: 24px; }
.section-emoji { font-size: 1.7rem; }
.section-name { font-family: 'Georgia', serif; font-size: 1.15rem; font-weight: 900; letter-spacing: 2px; text-transform: uppercase; color: var(--text); }
.section-line { flex: 1; height: 2px; background: linear-gradient(90deg, var(--border) 0%, transparent 100%); }
.section-count { background: linear-gradient(135deg, var(--red) 0%, var(--red2) 100%); color: #fff; font-size: .75rem; font-weight: 900; padding: 3px 10px; border-radius: 999px; font-family: sans-serif; }
.product-grid { display: flex; flex-wrap: wrap; gap: 18px; }
@media (max-width: 600px) {
  .product-grid { gap: 12px; }
  .product-card { max-width: calc((100% - 12px) / 2); }
}

/* CARD CON IMAGEN */
.product-card {
  background: var(--card); border: 2px solid var(--border); border-radius: var(--radius);
  overflow: hidden; cursor: pointer; display: flex; flex-direction: column;
  flex: 1 1 200px; max-width: calc((100% - 3 * 18px) / 4);
  transition: transform .25s, box-shadow .25s, border-color .25s;
  animation: fadeUp .45s ease both; box-shadow: 0 2px 12px rgba(0,0,0,.07);
}
@keyframes fadeUp { from{opacity:0;transform:translateY(14px)} to{opacity:1;transform:none} }
.product-card:hover { transform: translateY(-6px); border-color: var(--red); box-shadow: 0 12px 36px rgba(200,39,30,.2); }
.card-img {
  aspect-ratio: 1/1; width: 100%; position: relative;
  border-bottom: 2px solid var(--border); overflow: hidden;
  background: linear-gradient(145deg, #FFF3E5, #FFE2C0);
}
.card-photo {
  width: 100%; height: 100%; object-fit: cover;
  transition: transform .4s ease; display: block;
}
.product-card:hover .card-photo { transform: scale(1.07); }
.hot-badge {
  position: absolute; top: 8px; right: 8px;
  background: linear-gradient(135deg, var(--red) 0%, #FF4B3E 100%);
  color: #fff; font-size: .6rem; font-weight: 900;
  padding: 4px 8px; border-radius: 6px; letter-spacing: .5px;
  border: 1.5px solid rgba(255,255,255,.5); font-family: sans-serif;
  box-shadow: 0 2px 8px rgba(200,39,30,.4);
}
.card-body { padding: 14px; flex: 1; display: flex; flex-direction: column; }
.card-name { font-family: 'Georgia', serif; font-size: .84rem; font-weight: 900; letter-spacing: .5px; text-transform: uppercase; color: var(--red); margin-bottom: 5px; line-height: 1.2; }
.card-desc { font-size: .75rem; color: var(--muted); line-height: 1.4; flex: 1; margin-bottom: 10px; font-weight: 600; font-family: sans-serif; }
.card-foot { display: flex; align-items: center; justify-content: space-between; gap: 6px; padding-top: 10px; border-top: 1.5px solid var(--border); }
.card-price { font-family: 'Georgia', serif; font-size: 1rem; font-weight: 900; color: var(--red); }
.add-btn {
  background: linear-gradient(135deg, var(--red) 0%, var(--red2) 100%);
  color: #fff; border: none; border-radius: var(--radius-sm);
  padding: 7px 11px; font-size: .72rem; font-weight: 800; letter-spacing: .5px;
  cursor: pointer; transition: all .18s; white-space: nowrap; font-family: sans-serif;
  box-shadow: 0 3px 10px rgba(200,39,30,.3);
}
.add-btn:hover { transform: scale(1.06); box-shadow: 0 5px 16px rgba(200,39,30,.45); }

/* QTY */
.qty-row { display: flex; align-items: center; gap: 4px; }
.qty-row.lg { gap: 8px; }
.qty-row.sm { gap: 3px; }
.qty-btn {
  width: 28px; height: 28px; background: var(--bg); border: 2px solid var(--red);
  border-radius: 7px; color: var(--red); font-size: .95rem; font-weight: 900;
  cursor: pointer; display: flex; align-items: center; justify-content: center;
  transition: all .15s; font-family: sans-serif;
}
.qty-btn.lg { width: 38px; height: 38px; font-size: 1.2rem; }
.qty-btn:hover { background: var(--red); color: #fff; }
.qty-val { font-family: 'Georgia', serif; font-size: .88rem; font-weight: 900; color: var(--red); min-width: 20px; text-align: center; }
.qty-val.lg { font-size: 1.1rem; min-width: 28px; }

/* ═══════ DETAIL MODAL — REDISEÑADO ═══════ */
.overlay {
  position: fixed; inset: 0; background: rgba(0,0,0,.88); z-index: 300;
  display: flex; align-items: center; justify-content: center; padding: 20px;
  backdrop-filter: blur(3px);
}
.detail-card {
  background: var(--card); border-radius: var(--radius); border: 2px solid var(--border);
  width: 100%; max-width: 520px; max-height: 90vh; overflow-y: auto;
  position: relative; animation: popIn .3s cubic-bezier(.34,1.56,.64,1);
  box-shadow: 0 24px 70px rgba(0,0,0,.4);
}
@keyframes popIn { from{opacity:0;transform:scale(.86)} to{opacity:1;transform:none} }
.detail-close {
  position: absolute; top: 14px; right: 14px; z-index: 10;
  background: rgba(255,255,255,.95); border: 1.5px solid var(--border);
  border-radius: 8px; width: 36px; height: 36px;
  font-size: .9rem; cursor: pointer; color: var(--muted);
  display: flex; align-items: center; justify-content: center; transition: all .18s;
  box-shadow: 0 2px 8px rgba(0,0,0,.15);
}
.detail-close:hover { border-color: var(--red); color: var(--red); }

/* Imagen del detalle */
.detail-img-wrap {
  position: relative; width: 100%; height: 260px; overflow: hidden;
  background: linear-gradient(145deg, #FFF3E5, #FFE2C0);
}
.detail-photo { width: 100%; height: 100%; object-fit: cover; display: block; }
.detail-img-overlay {
  position: absolute; bottom: 0; left: 0; right: 0; height: 80px;
  background: linear-gradient(transparent, rgba(0,0,0,.4));
}
.detail-popular-badge {
  position: absolute; top: 14px; left: 14px;
  background: linear-gradient(135deg, var(--red) 0%, #FF4B3E 100%);
  color: #fff; font-size: .75rem; font-weight: 900;
  padding: 6px 12px; border-radius: 8px; letter-spacing: .5px;
  border: 1.5px solid rgba(255,255,255,.4); font-family: sans-serif;
  box-shadow: 0 2px 10px rgba(200,39,30,.4);
}

.detail-body { padding: 24px 28px 28px; }
.detail-category-tag {
  display: inline-block; background: #FFF0E0; border: 1.5px solid #F5C48E;
  border-radius: 999px; padding: 4px 14px;
  font-size: .72rem; font-weight: 800; color: #A05010;
  letter-spacing: 1px; font-family: sans-serif; margin-bottom: 12px;
  text-transform: uppercase;
}
.detail-name {
  font-family: 'Georgia', serif; font-size: 1.6rem; font-weight: 900;
  color: var(--text); line-height: 1.2; margin-bottom: 12px;
}
.detail-desc {
  font-size: .95rem; color: #5A4A40; line-height: 1.75;
  margin-bottom: 20px; font-family: sans-serif; font-weight: 500;
}

/* Highlights del producto */
.detail-highlights {
  background: #F8F2EC; border-radius: var(--radius-sm);
  border-left: 4px solid var(--red);
  padding: 14px 18px; margin-bottom: 22px;
  display: grid; grid-template-columns: 1fr 1fr; gap: 7px;
}
.detail-highlight-item {
  display: flex; align-items: center; gap: 8px;
  font-size: .82rem; color: #3D2010; font-weight: 700; font-family: sans-serif;
}
.highlight-dot { color: var(--red); font-size: .7rem; flex-shrink: 0; }

.detail-foot {
  display: flex; align-items: center; justify-content: space-between;
  gap: 14px; padding-top: 20px; border-top: 2px solid var(--border);
}
.detail-price-block { display: flex; flex-direction: column; }
.detail-price-label { font-size: .7rem; font-weight: 800; color: var(--muted); text-transform: uppercase; letter-spacing: 1.5px; font-family: sans-serif; margin-bottom: 2px; }
.detail-price { font-family: 'Georgia', serif; font-size: 2rem; font-weight: 900; color: var(--red); line-height: 1; }
.cta-btn {
  background: linear-gradient(135deg, var(--red) 0%, var(--red2) 100%);
  color: #fff; border: none; border-radius: var(--radius-sm);
  padding: 13px 22px; font-family: sans-serif;
  font-size: .85rem; letter-spacing: 1px; font-weight: 800; text-transform: uppercase;
  cursor: pointer; transition: all .2s; box-shadow: 0 4px 14px rgba(200,39,30,.4);
  white-space: nowrap;
}
.cta-btn:hover { transform: scale(1.05); box-shadow: 0 6px 20px rgba(200,39,30,.55); }

/* ═══════ CART SIDEBAR ═══════ */
.overlay-bg { position: fixed; inset: 0; background: rgba(0,0,0,.75); z-index: 200; backdrop-filter: blur(2px); }
.cart-sidebar {
  position: fixed; top: 0; right: 0; width: 400px; max-width: 100vw;
  height: 100vh; background: var(--card); border-left: 2px solid var(--border);
  z-index: 201; display: flex; flex-direction: column;
  box-shadow: -10px 0 40px rgba(0,0,0,.2);
}
.sidebar-head {
  padding: 20px 22px; border-bottom: 2px solid var(--border);
  display: flex; align-items: center; justify-content: space-between;
  background: linear-gradient(135deg, #1C0A05 0%, #2D1008 100%);
}
.sidebar-head h2 { font-family: 'Georgia', serif; font-size: 1.1rem; font-weight: 900; letter-spacing: 1px; color: #fff; }
.sidebar-head .x-btn { background: rgba(255,255,255,.1); border-color: rgba(255,255,255,.2); color: #fff; }
.sidebar-head .x-btn:hover { background: rgba(255,255,255,.25); }
.sidebar-body { flex: 1; overflow-y: auto; padding: 16px; }
.empty-cart { text-align: center; padding: 70px 20px; }
.empty-emoji { font-size: 4rem; margin-bottom: 14px; }
.empty-msg { font-family: 'Georgia', serif; font-size: .95rem; color: var(--muted); font-weight: 900; letter-spacing: 1px; }
.empty-sub { font-size: .82rem; color: var(--muted); margin-top: 6px; font-family: sans-serif; }

/* Cart item con imagen */
.cart-item {
  background: var(--bg); border: 1.5px solid var(--border);
  border-radius: var(--radius-sm); padding: 12px;
  display: flex; align-items: center; gap: 10px; margin-bottom: 10px;
  transition: box-shadow .2s;
}
.cart-item:hover { box-shadow: 0 4px 16px rgba(0,0,0,.1); }
.ci-img-wrap {
  width: 56px; height: 56px; border-radius: 8px; overflow: hidden;
  background: linear-gradient(145deg, #FFF3E5, #FFE2C0); flex-shrink: 0;
  border: 1.5px solid var(--border);
}
.ci-img { width: 100%; height: 100%; object-fit: cover; display: block; }
.ci-info { flex: 1; min-width: 0; }
.ci-name { font-size: .78rem; font-weight: 900; color: var(--text); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; margin-bottom: 6px; font-family: sans-serif; }
.ci-row { display: flex; align-items: center; justify-content: space-between; }
.ci-total { font-family: 'Georgia', serif; font-size: .9rem; font-weight: 900; color: var(--red); }
.del-btn { background: none; border: none; cursor: pointer; font-size: 1rem; color: var(--muted); padding: 4px; border-radius: 4px; transition: color .2s; }
.del-btn:hover { color: var(--red); }
.sidebar-foot { padding: 18px 22px; border-top: 2px solid var(--border); background: var(--bg); }
.summary-line { display: flex; justify-content: space-between; font-size: .88rem; color: var(--muted); margin-bottom: 8px; font-weight: 700; font-family: sans-serif; }
.summary-line.total { font-family: 'Georgia', serif; font-size: 1.08rem; color: var(--red); border-top: 2px solid var(--border); padding-top: 10px; margin-top: 4px; font-weight: 900; }
.free { color: #16a34a !important; font-weight: 900; }
.checkout-btn {
  width: 100%; margin-top: 12px;
  background: linear-gradient(135deg, var(--red) 0%, var(--red2) 100%);
  color: #fff; border: none; border-radius: var(--radius-sm); padding: 14px; font-family: sans-serif;
  font-size: .88rem; letter-spacing: 2px; font-weight: 900; text-transform: uppercase;
  cursor: pointer; transition: all .2s; box-shadow: 0 4px 18px rgba(200,39,30,.4);
}
.checkout-btn:hover { transform: translateY(-2px); box-shadow: 0 8px 26px rgba(200,39,30,.55); }

/* ═══════ CHECKOUT MODAL ═══════ */
.checkout-modal {
  background: var(--card); border-radius: var(--radius); border: 2px solid var(--border);
  width: 100%; max-width: 580px; max-height: 92vh; overflow-y: auto;
  animation: popIn .3s cubic-bezier(.34,1.56,.64,1); box-shadow: var(--shadow-lg);
}
.modal-head {
  background: linear-gradient(135deg, #1C0A05 0%, #2D1008 100%);
  color: #fff; padding: 18px 22px;
  display: flex; align-items: center; justify-content: space-between;
  position: sticky; top: 0; z-index: 10;
  border-radius: calc(var(--radius) - 2px) calc(var(--radius) - 2px) 0 0;
}
.modal-head h2 { font-family: 'Georgia', serif; font-size: 1.1rem; font-weight: 900; letter-spacing: 1px; margin: 0; }
.modal-body { padding: 22px; }
.invoice-body { padding: 22px; background: #F9F3EE; }
.modal-foot { padding: 0 22px 22px; display: flex; gap: 10px; flex-wrap: wrap; }
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; }
@media (max-width: 480px) { .form-row { grid-template-columns: 1fr; } }
.form-group { margin-bottom: 14px; }
.form-group label { display: block; font-size: .72rem; font-weight: 900; letter-spacing: 1.5px; text-transform: uppercase; color: var(--red); margin-bottom: 6px; font-family: sans-serif; }
.form-group input, .form-group textarea, .form-select {
  width: 100%; border: 2px solid var(--border); border-radius: var(--radius-sm);
  background: var(--bg); padding: 10px 13px; font-size: .92rem; font-weight: 600;
  color: var(--text); outline: none; transition: all .2s; font-family: sans-serif;
}
.form-group input:focus, .form-group textarea:focus, .form-select:focus {
  border-color: var(--red); background: #fff; box-shadow: 0 0 0 4px rgba(200,39,30,.08);
}
.form-group input.err, .form-select.err { border-color: #dc2626; background: #fef2f2; }
.err-msg { font-size: .74rem; color: #dc2626; font-weight: 800; margin-top: 4px; display: block; font-family: sans-serif; }
.form-select {
  appearance: none; cursor: pointer; padding-right: 32px;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='8' viewBox='0 0 12 8'%3E%3Cpath fill='%23C8271E' d='M6 8L0 0h12z'/%3E%3C/svg%3E");
  background-repeat: no-repeat; background-position: right 12px center;
}
textarea { resize: vertical; min-height: 72px; }
.btn-sec {
  flex: 1; background: var(--bg); border: 2px solid var(--border);
  border-radius: var(--radius-sm); padding: 12px; font-family: sans-serif;
  font-size: .8rem; font-weight: 800; letter-spacing: 1px; color: var(--muted);
  cursor: pointer; transition: all .2s; text-transform: uppercase;
}
.btn-sec:hover { border-color: var(--red); color: var(--red); }
.btn-pri {
  flex: 2; background: linear-gradient(135deg, var(--red) 0%, var(--red2) 100%); color: #fff;
  border: none; border-radius: var(--radius-sm); padding: 12px; font-family: sans-serif;
  font-size: .8rem; font-weight: 800; letter-spacing: 1px; cursor: pointer;
  transition: all .2s; text-transform: uppercase; box-shadow: 0 4px 14px rgba(200,39,30,.35);
}
.btn-pri:hover { transform: translateY(-1px); box-shadow: 0 6px 20px rgba(200,39,30,.5); }
.btn-pdf {
  flex: 1.3; background: #2D1008; color: #fff; border: none; border-radius: var(--radius-sm);
  padding: 12px; font-family: sans-serif; font-size: .78rem; font-weight: 800; letter-spacing: 1px;
  cursor: pointer; transition: all .2s; text-transform: uppercase;
}
.btn-pdf:hover { background: #1C0A05; transform: translateY(-1px); }

/* ═══════ FACTURA REDISEÑADA — MÁXIMA LEGIBILIDAD ═══════ */
.invoice {
  background: #fff;
  border-radius: 12px;
  border: 2px solid #E0D0C0;
  overflow: hidden;
  box-shadow: 0 4px 20px rgba(0,0,0,.1);
}

/* Header de la factura */
.inv-header {
  background: #1C0A05;
  padding: 20px 24px 16px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 12px;
}
.inv-logo-section {
  display: flex; align-items: center; gap: 12px;
}
.inv-fire-icon { font-size: 2rem; filter: drop-shadow(0 0 8px rgba(255,150,50,.7)); }
.inv-brand-name {
  font-family: 'Georgia', serif;
  font-size: 1.3rem; font-weight: 900; color: #fff;
  letter-spacing: 2px; text-transform: uppercase; line-height: 1;
}
.inv-brand-sub {
  font-size: .7rem; color: rgba(255,210,150,.75);
  letter-spacing: 1px; font-family: sans-serif; margin-top: 3px;
}
.inv-meta {
  display: flex; gap: 20px;
}
.inv-meta-item {
  display: flex; flex-direction: column; align-items: flex-end; gap: 2px;
}
.inv-meta-label {
  font-size: .62rem; font-weight: 900; letter-spacing: 2px;
  text-transform: uppercase; color: rgba(255,200,100,.6); font-family: sans-serif;
}
.inv-meta-value {
  font-family: 'Georgia', serif; font-size: .92rem; font-weight: 900; color: #fff;
}
.inv-num { color: var(--gold); font-size: 1.05rem; }

/* Secciones de la factura */
.inv-section {
  border-bottom: 1.5px solid #EAE0D5;
  padding: 16px 20px;
}
.inv-section-title {
  display: flex; align-items: center; gap: 7px;
  font-size: .68rem; font-weight: 900; letter-spacing: 2px;
  text-transform: uppercase; color: #8B5C30; margin-bottom: 12px;
  font-family: sans-serif;
}
.inv-section-icon { font-size: .9rem; }

/* Grid de datos del cliente */
.inv-client-grid {
  display: grid; grid-template-columns: 1fr 1fr; gap: 10px;
}
.inv-field {
  display: flex; flex-direction: column; gap: 2px;
}
.inv-field-full { grid-column: span 2; }
.inv-field-label {
  font-size: .64rem; font-weight: 800; letter-spacing: 1px;
  text-transform: uppercase; color: #9A7A60; font-family: sans-serif;
}
.inv-field-value {
  font-size: .88rem; font-weight: 700; color: #1C1410;
  font-family: sans-serif; line-height: 1.3;
}

/* Lista de productos */
.inv-products-list { }
.inv-product-header {
  display: grid; grid-template-columns: 1fr 60px 80px 90px;
  background: #1C0A05; padding: 8px 14px;
  font-size: .64rem; font-weight: 900; letter-spacing: 1.5px;
  text-transform: uppercase; color: rgba(255,210,150,.85); font-family: sans-serif;
}
.iph-name { }
.iph-qty, .iph-unit, .iph-total { text-align: center; }
.iph-total { text-align: right; }

.inv-product-row {
  display: grid; grid-template-columns: 1fr 60px 80px 90px;
  padding: 10px 14px; border-bottom: 1px solid #F0E8E0;
  align-items: center;
}
.inv-row-alt { background: #FDF8F3; }
.ipr-name {
  display: flex; align-items: center; gap: 7px;
  font-size: .85rem; font-weight: 700; color: #1C1410; font-family: sans-serif;
}
.ipr-emoji { font-size: 1.1rem; }
.ipr-qty { text-align: center; }
.qty-badge {
  display: inline-flex; align-items: center; justify-content: center;
  background: var(--red); color: #fff;
  border-radius: 999px; padding: 2px 10px;
  font-size: .78rem; font-weight: 900; font-family: sans-serif;
  min-width: 26px;
}
.ipr-unit {
  text-align: center; font-size: .82rem; color: #6A5040;
  font-weight: 600; font-family: sans-serif;
}
.ipr-total {
  text-align: right; font-size: .9rem; font-weight: 900;
  color: #1C1410; font-family: 'Georgia', serif;
}

/* Totales de la factura */
.inv-totals-section {
  padding: 16px 20px;
  background: #FFFCF8;
  border-bottom: 1.5px solid #EAE0D5;
}
.inv-total-row {
  display: flex; justify-content: space-between; align-items: center;
  padding: 5px 0;
}
.itr-label { font-size: .87rem; color: #6A5040; font-weight: 700; font-family: sans-serif; }
.itr-value { font-size: .9rem; font-weight: 800; color: #1C1410; font-family: sans-serif; }
.inv-row-green .itr-label, .inv-row-green .itr-value { color: #16a34a; }
.inv-row-cambio .itr-label, .inv-row-cambio .itr-value { color: #16a34a; }

.inv-grand-total {
  display: flex; justify-content: space-between; align-items: center;
  background: #1C0A05; color: #fff;
  border-radius: 8px; padding: 14px 18px; margin-top: 12px;
}
.igt-label {
  font-size: .82rem; font-weight: 800; letter-spacing: 1px;
  text-transform: uppercase; font-family: sans-serif; color: rgba(255,220,160,.9);
}
.igt-amount {
  font-family: 'Georgia', serif; font-size: 1.35rem; font-weight: 900; color: var(--gold);
}

/* Notas */
.inv-notes {
  display: flex; align-items: flex-start; gap: 10px;
  padding: 12px 20px; background: #FFFDE0;
  border-bottom: 1.5px solid #F5E090;
  font-size: .83rem; color: #5A4A10; font-family: sans-serif; line-height: 1.5;
}
.inv-notes strong { font-weight: 900; color: #3A3000; margin-right: 4px; }
.inv-notes-icon { font-size: 1rem; flex-shrink: 0; }

/* Footer de la factura */
.inv-footer-bar {
  background: #1C0A05;
  display: flex; justify-content: space-between; align-items: center;
  padding: 12px 20px;
  font-size: .75rem; font-weight: 800; letter-spacing: 2px;
  text-transform: uppercase; color: rgba(255,210,150,.7); font-family: sans-serif;
}
.inv-footer-phone { color: var(--gold); }

/* ═══════ SUCCESS ═══════ */
.success-box { padding: 56px 28px; text-align: center; display: flex; flex-direction: column; align-items: center; }
.success-icon { font-size: 4.5rem; margin-bottom: 18px; animation: popIn .5s ease; }
.success-title { font-family: 'Georgia', serif; font-size: 1.5rem; color: #16a34a; font-weight: 900; text-transform: uppercase; letter-spacing: 1.5px; margin-bottom: 8px; }
.success-sub { color: var(--muted); font-size: .95rem; margin-bottom: 22px; font-family: sans-serif; }
.order-chip { border: 3px dashed var(--red); border-radius: 12px; padding: 13px 22px; font-family: 'Georgia', serif; font-size: 1.15rem; color: var(--red); font-weight: 900; letter-spacing: 2.5px; background: var(--bg); margin-bottom: 16px; }
.success-time { color: var(--red); font-weight: 700; font-size: .92rem; margin-bottom: 28px; font-family: sans-serif; }

/* ═══════ ADMIN MODAL — REDISEÑADO ═══════ */
.admin-modal {
  background: #FAFAF8; border-radius: var(--radius); border: 2px solid var(--border);
  width: 100%; max-width: 860px; max-height: 92vh; overflow-y: auto;
  animation: popIn .3s cubic-bezier(.34,1.56,.64,1); box-shadow: var(--shadow-lg);
}

/* Header admin */
.admin-header {
  background: #fff; border-bottom: 2px solid var(--border);
  padding: 20px 28px;
  display: flex; align-items: center; gap: 16px;
  position: sticky; top: 0; z-index: 10;
  border-radius: calc(var(--radius) - 2px) calc(var(--radius) - 2px) 0 0;
}
.admin-header-left { display: flex; align-items: center; gap: 14px; flex: 1; }
.admin-header-icon {
  width: 48px; height: 48px; background: #FFF0E0;
  border: 2px solid #F5C48E; border-radius: 12px;
  display: flex; align-items: center; justify-content: center; font-size: 1.4rem;
  flex-shrink: 0;
}
.admin-header-title {
  font-family: 'Georgia', serif; font-size: 1.2rem; font-weight: 900;
  color: var(--text); margin: 0; letter-spacing: .5px;
}
.admin-header-sub {
  font-size: .78rem; color: var(--muted); font-family: sans-serif;
  margin-top: 2px; font-weight: 600;
}
.admin-close-btn {
  width: 38px; height: 38px; background: #F5EEE8; border: 1.5px solid var(--border);
  border-radius: 8px; font-size: .9rem; cursor: pointer; color: var(--muted);
  display: flex; align-items: center; justify-content: center; transition: all .18s;
  flex-shrink: 0; font-family: sans-serif;
}
.admin-close-btn:hover { border-color: var(--red); color: var(--red); background: #FFF0EE; }

/* Cuerpo admin en dos columnas */
.admin-body {
  display: grid; grid-template-columns: 1fr 320px;
  gap: 0; min-height: 400px;
}
@media (max-width: 700px) {
  .admin-body { grid-template-columns: 1fr; }
  .admin-preview-col { border-left: none; border-top: 2px solid var(--border); }
}

/* Columna formulario */
.admin-form-col { padding: 24px 28px; background: #fff; }
.admin-fieldset { margin-bottom: 24px; }
.admin-fieldset-title {
  font-size: .72rem; font-weight: 900; letter-spacing: 2px;
  text-transform: uppercase; color: var(--muted);
  margin-bottom: 14px; padding-bottom: 8px;
  border-bottom: 1.5px solid var(--border); font-family: sans-serif;
}
.admin-field { margin-bottom: 14px; }
.admin-label {
  display: block; font-size: .78rem; font-weight: 800;
  color: var(--text); margin-bottom: 7px; font-family: sans-serif;
  letter-spacing: .3px;
}
.req { color: var(--red); font-weight: 900; }
.admin-input {
  width: 100%; border: 1.5px solid #D5C8C0; border-radius: 8px;
  background: #FDFAF8; padding: 10px 14px;
  font-size: .9rem; font-weight: 600; color: var(--text);
  outline: none; transition: all .2s; font-family: sans-serif;
}
.admin-input:focus {
  border-color: var(--red); background: #fff;
  box-shadow: 0 0 0 3px rgba(200,39,30,.1);
}
.admin-input::placeholder { color: #B8A9A0; font-weight: 500; }
.admin-textarea {
  width: 100%; border: 1.5px solid #D5C8C0; border-radius: 8px;
  background: #FDFAF8; padding: 10px 14px;
  font-size: .9rem; font-weight: 600; color: var(--text);
  outline: none; transition: all .2s; font-family: sans-serif;
  resize: vertical; min-height: 90px;
}
.admin-textarea:focus { border-color: var(--red); background: #fff; box-shadow: 0 0 0 3px rgba(200,39,30,.1); }
.admin-textarea::placeholder { color: #B8A9A0; font-weight: 500; }
.admin-field-hint { display: block; font-size: .72rem; color: var(--muted); margin-top: 5px; font-family: sans-serif; }
.admin-select {
  width: 100%; border: 1.5px solid #D5C8C0; border-radius: 8px;
  background: #FDFAF8; padding: 10px 40px 10px 14px;
  font-size: .9rem; font-weight: 600; color: var(--text);
  outline: none; appearance: none; cursor: pointer; transition: all .2s; font-family: sans-serif;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='8' viewBox='0 0 12 8'%3E%3Cpath fill='%23C8271E' d='M6 8L0 0h12z'/%3E%3C/svg%3E");
  background-repeat: no-repeat; background-position: right 12px center;
}
.admin-select:focus { border-color: var(--red); background-color: #fff; box-shadow: 0 0 0 3px rgba(200,39,30,.1); }
.admin-two-col { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; }
.admin-price-wrap { position: relative; }
.admin-price-sym {
  position: absolute; left: 13px; top: 50%; transform: translateY(-50%);
  font-weight: 900; color: var(--red); font-family: 'Georgia', serif; font-size: 1rem;
  pointer-events: none;
}
.price-padded { padding-left: 26px !important; }
.emoji-centered { font-size: 1.5rem !important; text-align: center !important; padding: 7px !important; }

/* Toggle popular */
.admin-toggle-btn {
  width: 100%; display: flex; align-items: center; gap: 10px;
  cursor: pointer; padding: 10px 14px;
  border: 1.5px solid #D5C8C0; border-radius: 8px;
  background: #FDFAF8; transition: all .2s; font-family: sans-serif;
}
.admin-toggle-btn:hover { border-color: var(--red); }
.admin-toggle-on { border-color: var(--red) !important; background: #FFF8F5 !important; }
.admin-toggle-track {
  width: 40px; height: 22px; border-radius: 999px;
  background: #D5C8C0; position: relative; transition: background .25s; flex-shrink: 0;
}
.admin-toggle-on .admin-toggle-track { background: var(--red); }
.admin-toggle-thumb {
  position: absolute; top: 3px; left: 3px; width: 16px; height: 16px; border-radius: 50%;
  background: #fff; transition: transform .25s; box-shadow: 0 1px 4px rgba(0,0,0,.2);
}
.admin-toggle-on .admin-toggle-thumb { transform: translateX(18px); }
.admin-toggle-text { font-size: .83rem; font-weight: 700; color: var(--text); }
.admin-toggle-on .admin-toggle-text { color: var(--red); }
.admin-error {
  display: flex; align-items: center; gap: 7px;
  background: #FEF2F2; border: 1.5px solid #FCA5A5;
  border-radius: 8px; padding: 10px 14px;
  font-size: .82rem; font-weight: 800; color: #991B1B; font-family: sans-serif;
  margin-top: 4px;
}

/* Columna preview */
.admin-preview-col {
  border-left: 2px solid var(--border);
  background: #F5EDE4; padding: 24px 20px;
  display: flex; flex-direction: column; gap: 20px;
}
.admin-preview-title {
  font-size: .7rem; font-weight: 900; letter-spacing: 2px;
  text-transform: uppercase; color: var(--muted); font-family: sans-serif;
}

/* Preview card */
.admin-preview-card {
  background: #fff; border: 2px solid var(--border);
  border-radius: var(--radius); overflow: hidden;
  box-shadow: 0 4px 16px rgba(0,0,0,.1);
}
.apc-img {
  aspect-ratio: 4/3; background: linear-gradient(145deg, #FFF3E5, #FFE2C0);
  display: flex; align-items: center; justify-content: center;
  position: relative; overflow: hidden; border-bottom: 2px solid var(--border);
}
.apc-photo { width: 100%; height: 100%; object-fit: cover; display: block; }
.apc-emoji { font-size: 4rem; }
.apc-badge {
  position: absolute; top: 8px; right: 8px;
  background: var(--red); color: #fff; font-size: .6rem; font-weight: 900;
  padding: 4px 8px; border-radius: 6px; font-family: sans-serif;
}
.apc-body { padding: 14px; }
.apc-name { font-family: 'Georgia', serif; font-size: .85rem; font-weight: 900; color: var(--red); text-transform: uppercase; margin-bottom: 4px; }
.apc-desc { font-size: .75rem; color: var(--muted); margin-bottom: 10px; font-family: sans-serif; line-height: 1.4; }
.apc-foot { display: flex; justify-content: space-between; align-items: center; padding-top: 8px; border-top: 1.5px solid var(--border); }
.apc-price { font-family: 'Georgia', serif; font-size: .95rem; font-weight: 900; color: var(--red); }
.apc-add-btn {
  background: var(--red); color: #fff; font-family: sans-serif;
  font-size: .7rem; font-weight: 800; padding: 6px 10px; border-radius: 8px;
}

/* Preview empty */
.admin-preview-empty {
  display: flex; flex-direction: column; align-items: center; justify-content: center;
  background: #fff; border: 2px dashed #D5C8C0; border-radius: var(--radius);
  padding: 40px 20px; text-align: center; flex: 1; min-height: 200px;
}
.ape-icon { font-size: 2.5rem; margin-bottom: 12px; opacity: .4; }
.admin-preview-empty p { font-size: .83rem; color: var(--muted); font-family: sans-serif; font-weight: 600; line-height: 1.5; }

/* Tips */
.admin-tips {
  background: #FFF8EE; border: 1.5px solid #F5C48E;
  border-radius: 10px; padding: 14px 16px;
}
.admin-tips-title {
  font-size: .7rem; font-weight: 900; letter-spacing: 1.5px; text-transform: uppercase;
  color: #9A5010; margin-bottom: 10px; font-family: sans-serif;
}
.admin-tips-list {
  list-style: none; display: flex; flex-direction: column; gap: 6px;
}
.admin-tips-list li {
  font-size: .78rem; color: #6A4020; font-family: sans-serif;
  font-weight: 600; line-height: 1.4;
  padding-left: 16px; position: relative;
}
.admin-tips-list li::before { content: '→'; position: absolute; left: 0; color: var(--red); font-weight: 900; }

/* Footer admin */
.admin-footer {
  display: flex; gap: 10px; padding: 18px 28px;
  border-top: 2px solid var(--border); background: #fff;
  border-radius: 0 0 calc(var(--radius) - 2px) calc(var(--radius) - 2px);
}
.admin-save-btn { flex: 2; }

/* ═══════ X BTN ═══════ */
.x-btn {
  background: var(--bg); border: 1.5px solid var(--border);
  border-radius: 8px; width: 36px; height: 36px;
  font-size: .9rem; cursor: pointer; color: var(--muted);
  display: flex; align-items: center; justify-content: center;
  transition: all .18s; flex-shrink: 0; font-family: sans-serif;
}
.x-btn:hover { border-color: var(--red); color: var(--red); }
.x-btn.dark { background: rgba(255,255,255,.15); border-color: rgba(255,255,255,.3); color: #fff; }
.x-btn.dark:hover { background: rgba(255,255,255,.3); border-color: #fff; }

/* ═══════ FOOTER ═══════ */
.footer {
  background: linear-gradient(135deg, #1C0A05 0%, #2D1008 100%);
  color: rgba(255,220,180,.6); padding: 20px 28px; border-top: 3px solid var(--red);
}
.footer-inner {
  max-width: 1400px; margin: 0 auto;
  display: flex; flex-wrap: wrap; align-items: center;
  gap: 8px; justify-content: center; font-size: .82rem; font-weight: 600; font-family: sans-serif;
}
.footer-logo { font-family: 'Georgia', serif; font-size: .95rem; font-weight: 900; color: var(--gold); letter-spacing: 1px; }
.footer-sep { opacity: .3; }

/* ═══════ SPINNER + TOASTS ═══════ */
.spinner-overlay {
  position: fixed; inset: 0; background: rgba(0,0,0,.95); z-index: 500;
  display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 20px;
}
.spinner-ring {
  width: 60px; height: 60px; border-radius: 50%;
  border: 4px solid rgba(255,255,255,.1);
  border-top-color: var(--red); border-right-color: var(--gold);
  animation: spin .7s linear infinite;
}
@keyframes spin { to{transform:rotate(360deg)} }
.spinner-text { font-family: 'Georgia', serif; font-size: 1.1rem; letter-spacing: 2.5px; color: #fff; font-weight: 900; text-transform: uppercase; animation: blink 1.2s ease infinite; }
@keyframes blink { 0%,100%{opacity:1} 50%{opacity:.3} }
.toast-container { position: fixed; top: 20px; right: 20px; z-index: 999; display: flex; flex-direction: column; gap: 8px; pointer-events: none; }
.toast {
  background: #2D1008; color: #fff; border-radius: var(--radius-sm); padding: 12px 18px;
  font-size: .87rem; font-weight: 700; display: flex; align-items: center; gap: 10px;
  min-width: 260px; pointer-events: all; box-shadow: 0 6px 24px rgba(0,0,0,.35);
  border: 1px solid rgba(255,255,255,.08); font-family: sans-serif;
  animation: toastIn .3s cubic-bezier(.34,1.56,.64,1);
}
.toast-dot { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; background: currentColor; }
.toast-success { background: #14532d; border-color: rgba(22,163,74,.4); }
.toast-error { background: #7f1d1d; border-color: rgba(220,38,38,.4); }
.toast-info { background: #78350f; border-color: rgba(245,158,11,.4); }
@keyframes toastIn { from{opacity:0;transform:translateX(60px)} to{opacity:1;transform:none} }

/* ═══════ TRANSITIONS ═══════ */
.fade-enter-active, .fade-leave-active { transition: opacity .2s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
.modal-enter-active, .modal-leave-active { transition: all .22s; }
.modal-enter-from, .modal-leave-to { opacity: 0; }
.cart-overlay-enter-active, .cart-overlay-leave-active { transition: opacity .25s; }
.cart-overlay-enter-from, .cart-overlay-leave-to { opacity: 0; }
.slide-right-enter-active, .slide-right-leave-active { transition: transform .28s ease; }
.slide-right-enter-from, .slide-right-leave-to { transform: translateX(100%); }
.toast-enter-active, .toast-leave-active { transition: all .25s; }
.toast-enter-from { opacity: 0; transform: translateX(60px); }
.toast-leave-to { opacity: 0; transform: translateX(60px); }

/* ═══════ RESPONSIVE ═══════ */
@media (max-width: 768px) {
  .cart-sidebar { width: 100vw; }
  .cart-label { display: none; }
  .cart-btn { padding: 11px 14px; }
  .hero-title { font-size: 2.5rem; }
  .modal-foot { flex-wrap: wrap; }
  .btn-pdf { flex: 1 0 100%; }
  .product-card { max-width: calc((100% - 18px) / 2) !important; }
  .detail-highlights { grid-template-columns: 1fr; }
  .inv-client-grid { grid-template-columns: 1fr; }
  .inv-field-full { grid-column: span 1; }
  .inv-product-header, .inv-product-row { grid-template-columns: 1fr 45px 65px 75px; }
  .admin-modal { max-width: 100%; }
}
@media (max-width: 480px) {
  .header-inner { padding: 14px 18px; }
  .hero { padding: 42px 18px 18px; }
  .products-main { padding: 26px 16px 56px; }
  .control-bar-inner { padding: 12px 18px; }
  .brand-name { font-size: 1.2rem; }
  .inv-product-header, .inv-product-row { grid-template-columns: 1fr 40px 60px; }
  .iph-unit, .ipr-unit { display: none; }
  .detail-body { padding: 18px 20px 22px; }
}
</style>