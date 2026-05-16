<script>
import { ref, computed, onMounted } from 'vue';
import { supabase } from './supabase';

function fmtNum(n){return n?.toLocaleString('id-ID')||'0'}
function timeAgo(ts){
  if(!ts) return '';
  const d = new Date(ts);
  const s = Math.floor((Date.now()-d.getTime())/1000);
  if(s<60) return 'just now';
  if(s<3600) return Math.floor(s/60)+'m ago';
  if(s<86400) return Math.floor(s/3600)+'h ago';
  return Math.floor(s/86400)+'d ago';
}

export default {
  setup() {
    const activeRole = ref('customer');
    const currentModule = ref('products');
    const toasts = ref([]);

    const products = ref([]);
    const productCategories = ref([]);
    const orders = ref([]);
    const orderItems = ref([]);
    const transactions = ref([]);
    const kitchenTickets = ref([]);
    const fulfillments = ref([]);
    const orderStatusLogs = ref([]);
    const users = ref([]);
    const recap = ref(null);

    const cart = ref([]);
    const orderType = ref('dine_in');
    const orderNotes = ref('');
    const deliveryAddress = ref('');
    const catFilter = ref('all');
    const orderFilter = ref('all');
    const paymentModal = ref(null);
    const paymentMethod = ref('cash');
    const amountTendered = ref(0);
    const orderDetailModal = ref(null);
    const addProductModal = ref(false);
    const newProduct = ref({name:'',category_id:'',price:0,emoji:'🍽️'});

    async function ensureTestUsers() {
      const { data: existingUsers } = await supabase.from('users').select('*');
      if (existingUsers && existingUsers.length > 0) {
        users.value = existingUsers;
        return;
      }
      const mockRoles = ['customer','cashier','kitchen','logistics','admin'];
      const mockNames = ['Budi Santoso','Siti Rahayu','Agus Prasetyo','Dewi Kurniawan','Admin'];
      const toInsert = mockRoles.map((r,i) => ({
        name: mockNames[i],
        email: r+'@bizcore.test',
        password: 'hash',
        role: r
      }));
      await supabase.from('users').insert(toInsert);
      const { data } = await supabase.from('users').select('*');
      users.value = data || [];
    }

    async function loadInitialData() {
      try {
        await ensureTestUsers();

        const [
          { data: cats },
          { data: prods },
          { data: ords },
          { data: items },
          { data: trans },
          { data: tickets },
          { data: fulfs },
          { data: logs }
        ] = await Promise.all([
          supabase.from('product_categories').select('*'),
          supabase.from('products').select('*'),
          supabase.from('orders').select('*, users(name)'),
          supabase.from('order_items').select('*'),
          supabase.from('transactions').select('*'),
          supabase.from('kitchen_tickets').select('*'),
          supabase.from('fulfillments').select('*'),
          supabase.from('order_status_log').select('*')
        ]);
        
        productCategories.value = cats || [];
        products.value = prods || [];
        
        orders.value = (ords || []).map(o => ({
          ...o,
          customerName: o.users?.name || 'Unknown',
          total: o.total_amount,
          orderType: o.order_type,
          createdAt: o.created_at,
          deliveryAddress: o.delivery_address,
          items: (items || []).filter(i => i.order_id === o.id).map(i => ({
            id: i.id, name: i.product_name, price: i.unit_price, qty: i.quantity
          }))
        })).sort((a,b)=>new Date(b.created_at)-new Date(a.created_at));

        transactions.value = (trans || []).map(t => ({
          ...t, receipt: t.receipt_number, createdAt: t.created_at, amount: t.amount_due
        }));

        kitchenTickets.value = (tickets || []).map(t => ({
          ...t, orderId: t.order_id, createdAt: t.created_at
        }));

        fulfillments.value = (fulfs || []).map(f => ({
          ...f, orderId: f.order_id, createdAt: f.created_at, address: f.delivery_address
        }));

        orderStatusLogs.value = (logs || []).map(l => ({
          ...l, orderId: l.order_id, to: l.to_status, at: l.created_at
        }));

      } catch(e) {
        console.error(e);
        toast('Failed to load data', 'error');
      }
    }

    onMounted(loadInitialData);

    const navItems = [
      {id:'products', label:'Products', icon:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><rect x="3" y="3" width="7" height="7" rx="1"/><rect x="14" y="3" width="7" height="7" rx="1"/><rect x="3" y="14" width="7" height="7" rx="1"/><rect x="14" y="14" width="7" height="7" rx="1"/></svg>'},
      {id:'orders', label:'Orders', icon:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><path d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2"/><rect x="9" y="3" width="6" height="4" rx="1"/><path d="M9 12h6M9 16h4"/></svg>', badge:'awaiting_payment'},
      {id:'cashier', label:'Cashier', icon:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><rect x="2" y="5" width="20" height="14" rx="2"/><path d="M2 10h20"/></svg>', badge:'cashier_pending'},
      {id:'kitchen', label:'Kitchen', icon:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><path d="M6 13.87A4 4 0 017.41 6a5.11 5.11 0 011.05-1.29A5 5 0 0116 6a4 4 0 011 7.87V21H7z"/><path d="M6 17h12"/></svg>', badge:'kitchen_active'},
      {id:'logistics', label:'Logistics', icon:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><path d="M5 17H3a2 2 0 01-2-2V5a2 2 0 012-2h11a2 2 0 012 2v3"/><rect x="9" y="11" width="14" height="10" rx="1"/><circle cx="12" cy="21" r="1"/><circle cx="20" cy="21" r="1"/></svg>', badge:'logistics_pending'},
      {id:'admin', label:'Admin', icon:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8"><circle cx="12" cy="8" r="4"/><path d="M3 20a9 9 0 0118 0"/></svg>'},
    ];

    function getBadgeCount(key){
      if(key==='awaiting_payment') return orders.value.filter(o=>o.status==='awaiting_payment').length||0;
      if(key==='cashier_pending') return orders.value.filter(o=>o.status==='awaiting_payment').length||0;
      if(key==='kitchen_active') return kitchenTickets.value.filter(t=>['queued','preparing'].includes(t.status)).length||0;
      if(key==='logistics_pending') return fulfillments.value.filter(f=>['pending','dispatched'].includes(f.status)).length||0;
      return 0;
    }

    const allowedModules = computed(()=>{
      const map = {
        customer:['products','orders'], cashier:['cashier','orders'], kitchen:['kitchen'], logistics:['logistics'], admin:['products','orders','cashier','kitchen','logistics','admin'],
      };
      return map[activeRole.value]||[];
    });

    const visibleNav = computed(()=> navItems.filter(n=>allowedModules.value.includes(n.id)));

    function onRoleChange(){
      const allowed = allowedModules.value;
      if(!allowed.includes(currentModule.value)) currentModule.value = allowed[0];
    }

    function switchModule(id){
      if(!allowedModules.value.includes(id)) return toast('Access denied for your role','error');
      currentModule.value = id;
    }

    const currentUser = computed(()=>{
      const u = users.value.find(u=>u.role === activeRole.value);
      if(!u) return { name:'Unknown', role:activeRole.value, initials:'?' };
      return { id: u.id, name: u.name, role: u.role, initials: u.name.split(' ').map(w=>w[0]).join('').slice(0,2).toUpperCase() };
    });

    const moduleTitles = {
      products:['Product Catalog','Public listing · categories · pricing'],
      orders:['Order Management','Order lifecycle tracking'],
      cashier:['Cashier / POS','Payment confirmation · Financial records'],
      kitchen:['Kitchen Display System','Ticket queue · preparation status'],
      logistics:['Logistics','Dispatch and delivery fulfillment'],
      admin:['Admin Dashboard','Full system visibility'],
    };
    const currentModuleTitle = computed(()=>moduleTitles[currentModule.value]?.[0]||'');
    const currentModuleDesc  = computed(()=>moduleTitles[currentModule.value]?.[1]||'');

    const categories = computed(()=> productCategories.value.map(c=>c.name));
    function getCategoryName(id) {
      return productCategories.value.find(c=>c.id===id)?.name || 'Unknown';
    }

    const filteredProducts = computed(()=>{
      const mapped = products.value.map(p => ({
        ...p,
        category: getCategoryName(p.category_id),
        price: p.base_price,
        available: p.is_available
      }));
      if(catFilter.value==='all') return mapped;
      return mapped.filter(p=>p.category===catFilter.value);
    });
    const canEdit = computed(()=>activeRole.value==='admin');

    function cartQty(pid){ return cart.value.find(i=>i.id===pid)?.qty||0; }
    function addToCart(p){
      const existing = cart.value.find(i=>i.id===p.id);
      if(existing) existing.qty++;
      else cart.value.push({id:p.id,name:p.name,price:p.price,qty:1});
      toast(`${p.name} added to cart`);
    }
    function changeQty(pid,delta){
      const i = cart.value.findIndex(c=>c.id===pid);
      if(i===-1) return;
      cart.value[i].qty += delta;
      if(cart.value[i].qty <= 0) cart.value.splice(i,1);
    }
    const cartTotal = computed(()=>cart.value.reduce((s,i)=>s+i.price*i.qty,0));

    async function submitOrder(){
      if(cart.value.length===0) return toast('Cart is empty','error');
      if(orderType.value==='delivery' && !deliveryAddress.value) return toast('Enter delivery address','error');
      
      const cUser = users.value.find(u=>u.role==='customer');
      if(!cUser) return toast('Customer user not found', 'error');

      const { data: orderData, error: orderErr } = await supabase.from('orders').insert({
        customer_id: cUser.id,
        order_type: orderType.value,
        total_amount: cartTotal.value,
        status: 'awaiting_payment',
        notes: orderNotes.value,
        delivery_address: deliveryAddress.value
      }).select().single();

      if(orderErr) return toast(orderErr.message, 'error');

      const orderId = orderData.id;
      const itemsToInsert = cart.value.map(c => ({
        order_id: orderId,
        product_id: c.id,
        product_name: c.name,
        unit_price: c.price,
        quantity: c.qty,
        subtotal: c.price * c.qty
      }));

      await supabase.from('order_items').insert(itemsToInsert);
      await supabase.from('order_status_log').insert({ order_id: orderId, from_status: 'created', to_status: 'awaiting_payment', actor_id: cUser.id });

      cart.value = [];
      orderNotes.value = '';
      deliveryAddress.value = '';
      toast('Order placed! Awaiting payment ✓','success');
      currentModule.value = 'orders';
      loadInitialData();
    }

    const orderStatusFilters = ['all','created','awaiting_payment','paid','processing','ready','dispatched','completed','cancelled'];
    const filteredOrders = computed(()=>{
      let list = activeRole.value==='customer'
        ? orders.value.filter(o=>o.customerName===currentUser.value.name)
        : orders.value;
      if(orderFilter.value!=='all') list = list.filter(o=>o.status===orderFilter.value);
      return list;
    });

    async function cancelOrder(o){
      await supabase.from('orders').update({status:'cancelled'}).eq('id', o.id);
      await supabase.from('order_status_log').insert({order_id: o.id, to_status: 'cancelled', actor_id: currentUser.value.id});
      toast('Order cancelled');
      loadInitialData();
    }
    function viewOrder(o){ orderDetailModal.value = o; }
    function getOrder(id){ return orders.value.find(o=>o.id===id); }
    function getOrderLog(id){ return orderStatusLogs.value.filter(l=>l.orderId===id).sort((a,b)=>new Date(a.at)-new Date(b.at)); }

    const pendingPaymentOrders = computed(()=>orders.value.filter(o=>o.status==='awaiting_payment'));
    const todayRevenue = computed(()=>transactions.value.reduce((s,t)=>s+parseFloat(t.amount),0));
    const avgOrderValue = computed(()=>transactions.value.length ? Math.round(todayRevenue.value/transactions.value.length) : 0);

    function openPayment(o){
      paymentModal.value = o;
      paymentMethod.value = 'cash';
      amountTendered.value = o.total;
    }

    async function confirmPayment(){
      const o = paymentModal.value;
      if(!o) return;

      const { data: transData, error } = await supabase.from('transactions').insert({
        order_id: o.id,
        cashier_id: currentUser.value.id,
        payment_method: paymentMethod.value,
        amount_tendered: amountTendered.value,
        amount_due: o.total,
        change_given: Math.max(0, amountTendered.value - o.total),
        status: 'completed',
        receipt_number: 'RCP' + Date.now().toString().slice(-6)
      });
      if(error) return toast(error.message, 'error');

      await supabase.from('orders').update({status: 'processing'}).eq('id', o.id);
      await supabase.from('order_status_log').insert({ order_id: o.id, from_status: 'awaiting_payment', to_status: 'processing', actor_id: currentUser.value.id });

      await supabase.from('kitchen_tickets').insert({ order_id: o.id, status: 'queued' });

      if(o.orderType==='delivery'){
        await supabase.from('fulfillments').insert({ order_id: o.id, status: 'pending', delivery_address: o.deliveryAddress });
      }

      paymentModal.value = null;
      toast('Payment confirmed ✓','success');
      loadInitialData();
    }

    async function generateRecap(){
      const methods = ['cash','card','qr','transfer'];
      const totals = {};
      methods.forEach(m=>totals[m]=transactions.value.filter(t=>t.payment_method===m).reduce((s,t)=>s+parseFloat(t.amount),0));
      recap.value = {
        date: new Date().toLocaleDateString('id-ID',{day:'numeric',month:'long',year:'numeric'}),
        totalOrders: transactions.value.length,
        cash: totals.cash,
        card: totals.card,
        qr: (totals.qr||0)+(totals.transfer||0),
        total: todayRevenue.value,
      };
      
      const today = new Date().toISOString().split('T')[0];
      await supabase.from('daily_recaps').upsert({
        recap_date: today,
        total_revenue: recap.value.total,
        total_orders: recap.value.totalOrders,
        cash_total: recap.value.cash,
        card_total: recap.value.card,
        qr_total: recap.value.qr,
        generated_by: currentUser.value.id
      });
      toast('Daily recap generated ✓','success');
    }

    async function updateTicket(t, newStatus){
      const o = getOrder(t.orderId);
      await supabase.from('kitchen_tickets').update({status: newStatus}).eq('id', t.id);
      
      if(o && newStatus==='ready'){
        await supabase.from('orders').update({status: 'ready'}).eq('id', o.id);
        await supabase.from('order_status_log').insert({order_id: o.id, to_status: 'ready', actor_id: currentUser.value.id});
        
        if(o.orderType !== 'delivery'){
          setTimeout(async () => {
            await supabase.from('orders').update({status: 'completed'}).eq('id', o.id);
            loadInitialData();
          }, 1500);
        }
        toast('Order ready! 🍽️','success');
      } else if(newStatus==='preparing'){
        toast('Started preparing order');
      }
      loadInitialData();
    }

    async function updateFulfillment(f, newStatus){
      const o = getOrder(f.orderId);
      await supabase.from('fulfillments').update({status: newStatus}).eq('id', f.id);

      if(newStatus==='dispatched'){
        if(o) await supabase.from('orders').update({status: 'dispatched'}).eq('id', o.id);
        toast('Order dispatched 🚚');
      }
      if(newStatus==='delivered'){
        if(o) await supabase.from('orders').update({status: 'completed'}).eq('id', o.id);
        toast('Delivery confirmed ✓','success');
      }
      loadInitialData();
    }

    async function openAddProduct(){ 
      if (productCategories.value.length === 0) {
        await supabase.from('product_categories').insert([
          {name: 'Food', slug: 'food'},
          {name: 'Drinks', slug: 'drinks'},
          {name: 'Snacks', slug: 'snacks'},
          {name: 'Desserts', slug: 'desserts'}
        ]);
        const { data } = await supabase.from('product_categories').select('*');
        productCategories.value = data || [];
      }
      addProductModal.value = true; 
      newProduct.value={name:'', category_id: productCategories.value[0]?.id || '', price:0, emoji:'🍽️'}; 
    }

    async function saveProduct(){
      if(!newProduct.value.name || !newProduct.value.price) return toast('Fill all fields','error');
      await supabase.from('products').insert({
        name: newProduct.value.name,
        category_id: newProduct.value.category_id,
        base_price: newProduct.value.price,
        emoji: newProduct.value.emoji,
        is_available: true
      });
      addProductModal.value=false;
      toast('Product added ✓','success');
      loadInitialData();
    }

    const roleBadgeClass = computed(()=>({
      customer:'badge-created', cashier:'badge-awaiting', kitchen:'badge-processing',
      logistics:'badge-dispatched', admin:'badge-paid'
    }[activeRole.value]||'badge-created'));

    function roleBadgeClassFor(role){
      return {customer:'badge-created',cashier:'badge-awaiting',kitchen:'badge-processing',logistics:'badge-dispatched',admin:'badge-paid'}[role]||'';
    }

    function fmt(n){ return fmtNum(n); }

    let toastId=0;
    function toast(msg, type=''){
      const id=++toastId;
      toasts.value.push({id,msg,type});
      setTimeout(()=>{ toasts.value=toasts.value.filter(t=>t.id!==id); },2800);
    }

    return {
      activeRole, currentModule, toasts, products, orders, transactions,
      kitchenTickets, fulfillments, orderStatusLogs, recap, cart, orderType,
      orderNotes, deliveryAddress, catFilter, orderFilter, paymentModal,
      paymentMethod, amountTendered, orderDetailModal, addProductModal, newProduct,
      users, navItems, visibleNav, allowedModules, currentUser, currentModuleTitle,
      currentModuleDesc, categories, filteredProducts, canEdit, cartQty, addToCart,
      changeQty, cartTotal, submitOrder, orderStatusFilters, filteredOrders,
      cancelOrder, viewOrder, getOrder, getOrderLog, pendingPaymentOrders, todayRevenue,
      avgOrderValue, openPayment, confirmPayment, generateRecap, updateTicket,
      updateFulfillment, openAddProduct, saveProduct, roleBadgeClass, roleBadgeClassFor,
      fmt, timeAgo, getBadgeCount, onRoleChange, switchModule, productCategories
    };
  }
}
</script>

<template>
<div id="app">

  <!-- ── Sidebar ── -->
  <aside class="sidebar">
    <div class="sidebar-logo">
      <div class="logo-mark">BizCore</div>
      <div class="logo-sub">v1.0 · simulation</div>
    </div>

    <div class="nav-section">Modules</div>

    <div v-for="item in navItems" :key="item.id"
         class="nav-item" :class="{active: currentModule===item.id}"
         @click="switchModule(item.id)">
      <span v-html="item.icon" class="icon"></span>
      {{ item.label }}
      <span v-if="item.badge && getBadgeCount(item.badge)" class="nav-badge">
        {{ getBadgeCount(item.badge) }}
      </span>
    </div>

    <div class="sidebar-footer">
      <div class="user-pill">
        <div class="user-avatar">{{ currentUser.initials }}</div>
        <div>
          <div class="user-name">{{ currentUser.name }}</div>
          <div class="user-role">{{ currentUser.role }}</div>
        </div>
      </div>
      <div class="role-switcher">
        <select class="role-select" v-model="activeRole" @change="onRoleChange">
          <option value="customer">👤 Customer</option>
          <option value="cashier">💳 Cashier</option>
          <option value="kitchen">🍳 Kitchen</option>
          <option value="logistics">🚚 Logistics</option>
          <option value="admin">🔑 Admin</option>
        </select>
      </div>
    </div>
  </aside>

  <!-- ── Main ── -->
  <main class="main">
    <header class="topbar">
      <div>
        <div class="topbar-title">{{ currentModuleTitle }}</div>
        <div class="topbar-sub">{{ currentModuleDesc }}</div>
      </div>
      <div class="topbar-actions">
        <span style="font-size:12px;color:var(--text3)">Role:</span>
        <span class="badge" :class="roleBadgeClass">{{ activeRole }}</span>
        <button class="btn btn-secondary btn-sm" @click="generateSampleData" v-if="activeRole==='admin'">
          ⚡ Reset demo data
        </button>
      </div>
    </header>

    <div class="content">

      <!-- ══ PRODUCT MODULE ══ -->
      <div v-if="currentModule==='products'">
        <div class="section-title">Product Catalog</div>
        <div class="section-sub">Browse available items · {{ products.filter(p=>p.available).length }} items available</div>

        <div class="filter-bar">
          <span class="filter-chip" :class="{active:catFilter==='all'}" @click="catFilter='all'">All</span>
          <span class="filter-chip" v-for="c in categories" :key="c"
                :class="{active:catFilter===c}" @click="catFilter=c">{{ c }}</span>
          <button v-if="canEdit" class="btn btn-primary btn-sm" style="margin-left:auto" @click="openAddProduct">
            + Add product
          </button>
        </div>

        <div v-if="currentModule==='products'" class="two-col">
          <div>
            <div class="product-grid">
              <div v-for="p in filteredProducts" :key="p.id"
                   class="product-card" :class="{'in-cart': cartQty(p.id)>0}"
                   @click="p.available && addToCart(p)">
                <div class="product-img">{{ p.emoji }}</div>
                <div class="product-body">
                  <div class="product-name">{{ p.name }}</div>
                  <div class="product-cat">{{ p.category }}</div>
                  <div class="product-footer">
                    <div class="product-price">Rp {{ fmt(p.price) }}</div>
                    <div v-if="cartQty(p.id)>0" class="product-qty-badge">{{ cartQty(p.id) }}</div>
                    <div v-else class="product-avail" :class="{out:!p.available}">
                      {{ p.available ? 'Available' : 'Out of stock' }}
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- Cart panel (customer only) -->
          <div v-if="activeRole==='customer'">
            <div class="cart-panel">
              <div class="card-header">
                <div>
                  <div class="card-title">Your cart</div>
                  <div class="card-sub">{{ cart.length }} items</div>
                </div>
              </div>

              <div v-if="cart.length===0" class="cart-empty">
                <div style="font-size:28px;margin-bottom:8px">🛒</div>
                Tap a product to add it
              </div>

              <div v-for="item in cart" :key="item.id" class="cart-item">
                <div class="cart-item-name">{{ item.name }}</div>
                <div class="cart-qty-ctrl">
                  <div class="qty-btn" @click="changeQty(item.id,-1)">−</div>
                  <span style="font-size:13px;font-weight:600;width:18px;text-align:center">{{ item.qty }}</span>
                  <div class="qty-btn" @click="changeQty(item.id,1)">+</div>
                </div>
                <div class="cart-item-price">Rp {{ fmt(item.price * item.qty) }}</div>
              </div>

              <div v-if="cart.length>0">
                <div class="cart-total">Rp {{ fmt(cartTotal) }}</div>
                <div class="divider"></div>
                <div class="form-group">
                  <label class="form-label">Order type</label>
                  <div class="tabs">
                    <div class="tab" :class="{active:orderType==='dine_in'}" @click="orderType='dine_in'">Dine-in</div>
                    <div class="tab" :class="{active:orderType==='takeaway'}" @click="orderType='takeaway'">Takeaway</div>
                    <div class="tab" :class="{active:orderType==='delivery'}" @click="orderType='delivery'">Delivery</div>
                  </div>
                </div>
                <div class="form-group" v-if="orderType==='delivery'">
                  <label class="form-label">Delivery address</label>
                  <input class="form-input" v-model="deliveryAddress" placeholder="Enter delivery address"/>
                </div>
                <div class="form-group">
                  <label class="form-label">Notes (optional)</label>
                  <textarea class="form-textarea" v-model="orderNotes" placeholder="e.g. no onion, extra spicy…" rows="2"></textarea>
                </div>
                <button class="btn btn-primary" style="width:100%" @click="submitOrder">
                  Place order · Rp {{ fmt(cartTotal) }}
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- ══ ORDER MODULE ══ -->
      <div v-if="currentModule==='orders'">
        <div class="section-title">Orders</div>
        <div class="section-sub">{{ activeRole==='customer' ? 'Your order history' : 'All orders · live feed' }}</div>

        <div class="stat-grid" v-if="activeRole!=='customer'">
          <div class="stat-card">
            <div class="stat-label">Total orders</div>
            <div class="stat-value">{{ orders.length }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Pending payment</div>
            <div class="stat-value">{{ orders.filter(o=>o.status==='awaiting_payment').length }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">In progress</div>
            <div class="stat-value">{{ orders.filter(o=>['paid','processing'].includes(o.status)).length }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Completed today</div>
            <div class="stat-value">{{ orders.filter(o=>o.status==='completed').length }}</div>
          </div>
        </div>

        <div class="filter-bar">
          <span v-for="s in orderStatusFilters" :key="s"
                class="filter-chip" :class="{active:orderFilter===s}"
                @click="orderFilter=s">{{ s==='all' ? 'All' : s }}</span>
        </div>

        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>Order ID</th>
                <th>Customer</th>
                <th>Items</th>
                <th>Total</th>
                <th>Type</th>
                <th>Status</th>
                <th>Time</th>
                <th>Actions</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="o in filteredOrders" :key="o.id" @click="viewOrder(o)" style="cursor:pointer">
                <td class="td-mono">#{{ o.id.slice(-6).toUpperCase() }}</td>
                <td>{{ o.customerName }}</td>
                <td>{{ o.items.length }} item{{ o.items.length!==1?'s':'' }}</td>
                <td class="td-mono">Rp {{ fmt(o.total) }}</td>
                <td>{{ o.orderType.replace('_',' ') }}</td>
                <td><span class="badge" :class="'badge-'+o.status">{{ o.status.replace('_',' ') }}</span></td>
                <td class="td-mono">{{ timeAgo(o.createdAt) }}</td>
                <td @click.stop>
                  <button v-if="o.status==='created' && activeRole==='customer'" class="btn btn-danger btn-sm" @click="cancelOrder(o)">Cancel</button>
                  <button v-if="o.status==='created' && activeRole!=='customer'" class="btn btn-secondary btn-sm" @click="viewOrder(o)">View</button>
                  <span v-if="!['created'].includes(o.status)" style="font-size:12px;color:var(--text3)">—</span>
                </td>
              </tr>
              <tr v-if="filteredOrders.length===0">
                <td colspan="8"><div class="empty"><div class="empty-icon">📋</div><div class="empty-text">No orders found</div></div></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- ══ CASHIER MODULE ══ -->
      <div v-if="currentModule==='cashier'">
        <div class="section-title">Cashier / POS</div>
        <div class="section-sub">Confirm payments · Generate daily recap</div>

        <div class="stat-grid">
          <div class="stat-card">
            <div class="stat-label">Pending payment</div>
            <div class="stat-value" style="color:var(--amber)">{{ pendingPaymentOrders.length }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Today's revenue</div>
            <div class="stat-value">Rp {{ fmt(todayRevenue) }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Transactions</div>
            <div class="stat-value">{{ transactions.length }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Avg order value</div>
            <div class="stat-value">Rp {{ fmt(avgOrderValue) }}</div>
          </div>
        </div>

        <div class="three-col">
          <!-- Pending payment orders -->
          <div class="card" style="grid-column:span 2">
            <div class="card-header">
              <div>
                <div class="card-title">Awaiting payment</div>
                <div class="card-sub">{{ pendingPaymentOrders.length }} order(s) need payment confirmation</div>
              </div>
            </div>
            <div v-if="pendingPaymentOrders.length===0" class="empty">
              <div class="empty-icon">✅</div>
              <div class="empty-text">All caught up</div>
            </div>
            <div v-for="o in pendingPaymentOrders" :key="o.id"
                 style="border:1px solid var(--border);border-radius:var(--radius);padding:14px;margin-bottom:10px">
              <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:10px">
                <div>
                  <span class="td-mono" style="font-size:13px;font-weight:600">#{{ o.id.slice(-6).toUpperCase() }}</span>
                  <span style="font-size:12px;color:var(--text3);margin-left:8px">{{ o.orderType.replace('_',' ') }} · {{ o.customerName }}</span>
                </div>
                <span style="font-family:'DM Mono',monospace;font-size:15px;font-weight:700">Rp {{ fmt(o.total) }}</span>
              </div>
              <div style="font-size:12px;color:var(--text2);margin-bottom:12px">
                <span v-for="(it,i) in o.items" :key="i">{{ it.qty }}× {{ it.name }}<span v-if="i<o.items.length-1">, </span></span>
              </div>
              <div style="display:flex;gap:8px">
                <button class="btn btn-success btn-sm" @click="openPayment(o)">💳 Confirm payment</button>
                <button class="btn btn-danger btn-sm" @click="cancelOrder(o)">Void</button>
              </div>
            </div>
          </div>

          <!-- Transaction log -->
          <div class="card">
            <div class="card-header">
              <div>
                <div class="card-title">Transactions</div>
                <div class="card-sub">Today's log</div>
              </div>
            </div>
            <div v-if="transactions.length===0" class="empty" style="padding:20px">
              <div class="empty-text">No transactions yet</div>
            </div>
            <div v-for="t in transactions.slice().reverse()" :key="t.id"
                 style="padding:10px 0;border-bottom:1px solid var(--border)">
              <div style="display:flex;justify-content:space-between;align-items:center">
                <span class="td-mono" style="font-size:11px;color:var(--text3)">#{{ t.receipt }}</span>
                <span style="font-family:'DM Mono',monospace;font-size:13px;font-weight:600">Rp {{ fmt(t.amount) }}</span>
              </div>
              <div style="font-size:11px;color:var(--text3);margin-top:2px">{{ t.method }} · {{ timeAgo(t.createdAt) }}</div>
            </div>
            <div class="divider"></div>
            <button class="btn btn-primary" style="width:100%;margin-top:4px" @click="generateRecap">
              📊 Generate daily recap
            </button>
          </div>
        </div>

        <!-- Recap -->
        <div v-if="recap" class="card" style="margin-top:20px">
          <div class="card-header">
            <div class="card-title">Daily Recap — {{ recap.date }}</div>
          </div>
          <div class="recap-row"><span class="recap-label">Total orders</span><span class="recap-val">{{ recap.totalOrders }}</span></div>
          <div class="recap-row"><span class="recap-label">Cash</span><span class="recap-val">Rp {{ fmt(recap.cash) }}</span></div>
          <div class="recap-row"><span class="recap-label">Card</span><span class="recap-val">Rp {{ fmt(recap.card) }}</span></div>
          <div class="recap-row"><span class="recap-label">QR / Transfer</span><span class="recap-val">Rp {{ fmt(recap.qr) }}</span></div>
          <div class="recap-row">
            <span class="recap-label recap-total">Total revenue</span>
            <span class="recap-val recap-total">Rp {{ fmt(recap.total) }}</span>
          </div>
        </div>
      </div>

      <!-- ══ KITCHEN MODULE ══ -->
      <div v-if="currentModule==='kitchen'">
        <div class="section-title">Kitchen Display</div>
        <div class="section-sub">Active tickets · No financial data visible</div>

        <div class="stat-grid">
          <div class="stat-card">
            <div class="stat-label">Queued</div>
            <div class="stat-value">{{ kitchenTickets.filter(t=>t.status==='queued').length }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Preparing</div>
            <div class="stat-value" style="color:var(--amber)">{{ kitchenTickets.filter(t=>t.status==='preparing').length }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Ready</div>
            <div class="stat-value" style="color:var(--green)">{{ kitchenTickets.filter(t=>t.status==='ready').length }}</div>
          </div>
        </div>

        <div v-if="kitchenTickets.length===0" class="empty">
          <div class="empty-icon">🍳</div>
          <div class="empty-text">No active tickets — kitchen is idle</div>
        </div>

        <div class="ticket-grid">
          <div v-for="t in kitchenTickets" :key="t.id"
               class="ticket-card" :class="t.status">
            <div style="display:flex;align-items:center;justify-content:space-between">
              <span class="badge" :class="'badge-'+t.status">{{ t.status }}</span>
              <span class="ticket-id">#{{ t.orderId.slice(-6).toUpperCase() }}</span>
            </div>
            <div style="font-size:12px;color:var(--text3);margin-top:6px;text-transform:capitalize">
              {{ getOrder(t.orderId)?.orderType.replace('_',' ') }}
            </div>
            <div class="ticket-items">
              <div class="ticket-item-row" v-for="(it,i) in getOrder(t.orderId)?.items" :key="i">
                <span>{{ it.qty }}×</span> {{ it.name }}
              </div>
            </div>
            <div v-if="getOrder(t.orderId)?.notes" style="font-size:12px;background:var(--amber-bg);color:var(--amber);padding:6px 8px;border-radius:6px;margin-bottom:8px">
              📝 {{ getOrder(t.orderId)?.notes }}
            </div>
            <div class="ticket-time">{{ timeAgo(t.createdAt) }}</div>
            <div style="display:flex;gap:6px;margin-top:10px">
              <button v-if="t.status==='queued'" class="btn btn-secondary btn-sm" style="flex:1" @click="updateTicket(t,'preparing')">
                Start
              </button>
              <button v-if="t.status==='preparing'" class="btn btn-success btn-sm" style="flex:1" @click="updateTicket(t,'ready')">
                Mark ready ✓
              </button>
              <div v-if="t.status==='ready'" style="font-size:12px;color:var(--green);font-weight:600;padding:5px 0">
                ✓ Ready for pickup
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- ══ LOGISTICS MODULE ══ -->
      <div v-if="currentModule==='logistics'">
        <div class="section-title">Logistics & Fulfillment</div>
        <div class="section-sub">Manage dispatching and delivery confirmation</div>

        <div class="stat-grid">
          <div class="stat-card">
            <div class="stat-label">Pending dispatch</div>
            <div class="stat-value">{{ fulfillments.filter(f=>f.status==='pending').length }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Dispatched</div>
            <div class="stat-value" style="color:var(--blue)">{{ fulfillments.filter(f=>f.status==='dispatched').length }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Delivered today</div>
            <div class="stat-value" style="color:var(--green)">{{ fulfillments.filter(f=>f.status==='delivered').length }}</div>
          </div>
        </div>

        <div v-if="fulfillments.length===0" class="empty">
          <div class="empty-icon">🚚</div>
          <div class="empty-text">No fulfillments pending</div>
        </div>

        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>Fulfillment ID</th>
                <th>Order</th>
                <th>Customer</th>
                <th>Type</th>
                <th>Address</th>
                <th>Status</th>
                <th>Actions</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="f in fulfillments" :key="f.id">
                <td class="td-mono">#{{ f.id.slice(-6).toUpperCase() }}</td>
                <td class="td-mono">#{{ f.orderId.slice(-6).toUpperCase() }}</td>
                <td>{{ getOrder(f.orderId)?.customerName }}</td>
                <td style="text-transform:capitalize">{{ getOrder(f.orderId)?.orderType.replace('_',' ') }}</td>
                <td style="font-size:12px;color:var(--text2)">{{ f.address || '— pickup —' }}</td>
                <td><span class="badge" :class="'badge-'+f.status">{{ f.status }}</span></td>
                <td>
                  <button v-if="f.status==='pending'" class="btn btn-secondary btn-sm" @click="updateFulfillment(f,'dispatched')">
                    🚚 Dispatch
                  </button>
                  <button v-if="f.status==='dispatched'" class="btn btn-success btn-sm" @click="updateFulfillment(f,'delivered')">
                    ✓ Delivered
                  </button>
                  <span v-if="f.status==='delivered'" style="font-size:12px;color:var(--green);font-weight:600">✓ Done</span>
                </td>
              </tr>
              <tr v-if="fulfillments.length===0">
                <td colspan="7"><div class="empty"><div class="empty-text">No fulfillments</div></div></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- ══ ADMIN MODULE ══ -->
      <div v-if="currentModule==='admin'">
        <div class="section-title">Admin Dashboard</div>
        <div class="section-sub">Full system overview · All modules visible</div>

        <div class="stat-grid">
          <div class="stat-card">
            <div class="stat-label">Total orders</div>
            <div class="stat-value">{{ orders.length }}</div>
            <div class="stat-delta">all time</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Revenue</div>
            <div class="stat-value">Rp {{ fmt(todayRevenue) }}</div>
            <div class="stat-delta">from paid orders</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Products</div>
            <div class="stat-value">{{ products.length }}</div>
            <div class="stat-delta">{{ products.filter(p=>p.available).length }} available</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Active tickets</div>
            <div class="stat-value">{{ kitchenTickets.filter(t=>t.status!=='ready').length }}</div>
            <div class="stat-delta">in kitchen</div>
          </div>
        </div>

        <!-- Order pipeline -->
        <div class="card" style="margin-bottom:20px">
          <div class="card-header"><div class="card-title">Order pipeline</div></div>
          <div style="display:flex;gap:8px;flex-wrap:wrap">
            <div v-for="s in ['created','awaiting_payment','paid','processing','ready','dispatched','completed','cancelled']"
                 :key="s" style="flex:1;min-width:80px;text-align:center;padding:12px 8px;border-radius:var(--radius);border:1px solid var(--border)">
              <div style="font-size:22px;font-weight:700;font-family:'DM Mono',monospace">{{ orders.filter(o=>o.status===s).length }}</div>
              <div style="font-size:10px;color:var(--text3);margin-top:4px;text-transform:capitalize">{{ s.replace('_',' ') }}</div>
            </div>
          </div>
        </div>

        <!-- Users table -->
        <div class="card">
          <div class="card-header">
            <div class="card-title">Users & roles</div>
          </div>
          <div class="table-wrap" style="border:none">
            <table>
              <thead>
                <tr>
                  <th>Name</th><th>Email</th><th>Role</th><th>Status</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="u in users" :key="u.id">
                  <td style="font-weight:500">{{ u.name }}</td>
                  <td class="td-mono">{{ u.email }}</td>
                  <td><span class="badge" :class="roleBadgeClassFor(u.role)">{{ u.role }}</span></td>
                  <td><span class="badge badge-completed">active</span></td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>

    </div><!-- /content -->
  </main>

  <!-- ── Payment modal ── -->
  <div v-if="paymentModal" class="modal-overlay" @click.self="paymentModal=null">
    <div class="modal">
      <div class="modal-title">Confirm payment</div>
      <div class="modal-sub">Order #{{ paymentModal.id.slice(-6).toUpperCase() }} · {{ paymentModal.customerName }}</div>
      <div style="font-size:22px;font-weight:700;font-family:'DM Mono',monospace;margin-bottom:16px">
        Rp {{ fmt(paymentModal.total) }}
      </div>
      <div class="form-group">
        <label class="form-label">Payment method</label>
        <select class="form-select" v-model="paymentMethod">
          <option value="cash">💵 Cash</option>
          <option value="card">💳 Card</option>
          <option value="qr">📱 QR / QRIS</option>
          <option value="transfer">🏦 Bank transfer</option>
        </select>
      </div>
      <div v-if="paymentMethod==='cash'" class="form-group">
        <label class="form-label">Amount tendered</label>
        <input class="form-input" type="number" v-model.number="amountTendered"
               :placeholder="paymentModal.total"/>
        <div v-if="amountTendered >= paymentModal.total" style="font-size:12px;color:var(--green);margin-top:4px">
          Change: Rp {{ fmt(amountTendered - paymentModal.total) }}
        </div>
      </div>
      <div class="modal-actions">
        <button class="btn btn-secondary" @click="paymentModal=null">Cancel</button>
        <button class="btn btn-primary" @click="confirmPayment">Confirm payment</button>
      </div>
    </div>
  </div>

  <!-- ── Order detail modal ── -->
  <div v-if="orderDetailModal" class="modal-overlay" @click.self="orderDetailModal=null">
    <div class="modal" style="max-width:500px">
      <div class="modal-title">Order #{{ orderDetailModal.id.slice(-6).toUpperCase() }}</div>
      <div class="modal-sub">{{ orderDetailModal.customerName }} · {{ orderDetailModal.orderType.replace('_',' ') }}</div>
      <div style="margin-bottom:16px">
        <span class="badge" :class="'badge-'+orderDetailModal.status">{{ orderDetailModal.status.replace('_',' ') }}</span>
      </div>
      <div v-for="it in orderDetailModal.items" :key="it.id"
           style="display:flex;justify-content:space-between;padding:8px 0;border-bottom:1px solid var(--border);font-size:13px">
        <span>{{ it.qty }}× {{ it.name }}</span>
        <span class="td-mono">Rp {{ fmt(it.price * it.qty) }}</span>
      </div>
      <div style="display:flex;justify-content:space-between;font-weight:700;font-family:'DM Mono',monospace;margin-top:12px;font-size:15px">
        <span>Total</span><span>Rp {{ fmt(orderDetailModal.total) }}</span>
      </div>
      <div v-if="orderDetailModal.notes" style="margin-top:12px;font-size:12px;background:var(--amber-bg);color:var(--amber);padding:8px;border-radius:6px">
        📝 {{ orderDetailModal.notes }}
      </div>

      <!-- Status timeline -->
      <div style="margin-top:20px">
        <div style="font-size:12px;font-weight:600;color:var(--text2);margin-bottom:12px">Status history</div>
        <ul class="timeline">
          <li v-for="(log, i) in getOrderLog(orderDetailModal.id)" :key="i" class="tl-item">
            <div class="tl-line"></div>
            <div class="tl-dot" :class="i===getOrderLog(orderDetailModal.id).length-1?'active':'done'"></div>
            <div>
              <div class="tl-label">{{ log.to.replace('_',' ') }}</div>
              <div class="tl-time">{{ timeAgo(log.at) }}</div>
            </div>
          </li>
        </ul>
      </div>

      <div class="modal-actions">
        <button class="btn btn-secondary" @click="orderDetailModal=null">Close</button>
      </div>
    </div>
  </div>

  <!-- ── Add product modal ── -->
  <div v-if="addProductModal" class="modal-overlay" @click.self="addProductModal=false">
    <div class="modal">
      <div class="modal-title">Add product</div>
      <div class="modal-sub">New item to the catalog</div>
      <div class="form-group">
        <label class="form-label">Name</label>
        <input class="form-input" v-model="newProduct.name" placeholder="Product name"/>
      </div>
      <div class="form-group">
        <label class="form-label">Category</label>
        <select class="form-select" v-model="newProduct.category_id">
          <option v-for="c in productCategories" :key="c.id" :value="c.id">{{ c.name }}</option>
        </select>
      </div>
      <div class="form-group">
        <label class="form-label">Price (Rp)</label>
        <input class="form-input" type="number" v-model.number="newProduct.price" placeholder="25000"/>
      </div>
      <div class="form-group">
        <label class="form-label">Emoji icon</label>
        <input class="form-input" v-model="newProduct.emoji" placeholder="🍕"/>
      </div>
      <div class="modal-actions">
        <button class="btn btn-secondary" @click="addProductModal=false">Cancel</button>
        <button class="btn btn-primary" @click="saveProduct">Add product</button>
      </div>
    </div>
  </div>

  <!-- ── Toasts ── -->
  <div class="toast-wrap">
    <div v-for="t in toasts" :key="t.id" class="toast" :class="t.type">{{ t.msg }}</div>
  </div>

</div>
</template>

