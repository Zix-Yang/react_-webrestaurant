// routes/admin.js
const express = require('express');
const router  = express.Router();
const pool    = require('../config/db');
const { requireAdmin } = require('../middleware/auth');

// 所有路由都需要管理员权限
router.use(requireAdmin);

// ══════════════════════════════════════
// 菜品管理
// ══════════════════════════════════════

// GET /api/admin/dishes
router.get('/dishes', async (req, res) => {
  try {
    const [rows] = await pool.query(`
      SELECT d.*, c.category_name
      FROM dishes d
      LEFT JOIN categories c ON d.category_id = c.category_id
      ORDER BY c.sort_order, d.dish_id
    `);
    res.json(rows);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// POST /api/admin/dishes — 新增菜品
router.post('/dishes', async (req, res) => {
  const { name, description, price, original_price, image_url, category_id, spicy_level, is_featured } = req.body;
  if (!name || !price) return res.status(400).json({ message: '菜名和价格必填' });
  try {
    const [result] = await pool.query(
      `INSERT INTO dishes (name, description, price, original_price, image_url, category_id, spicy_level, is_featured)
       VALUES (?, ?, ?, ?, ?, ?, ?, ?)`,
      [name, description || '', price, original_price || null, image_url || null,
       category_id || null, spicy_level || 0, is_featured || false]
    );
    res.status(201).json({ message: '菜品添加成功', dish_id: result.insertId });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// PUT /api/admin/dishes/:id — 编辑菜品
router.put('/dishes/:id', async (req, res) => {
  const { name, description, price, original_price, image_url, category_id, spicy_level, is_featured, is_available } = req.body;
  try {
    await pool.query(
      `UPDATE dishes SET name=?, description=?, price=?, original_price=?, image_url=?,
       category_id=?, spicy_level=?, is_featured=?, is_available=? WHERE dish_id=?`,
      [name, description, price, original_price || null, image_url || null,
       category_id, spicy_level || 0, is_featured || false, is_available !== false, req.params.id]
    );
    res.json({ message: '更新成功' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// DELETE /api/admin/dishes/:id
router.delete('/dishes/:id', async (req, res) => {
  try {
    await pool.query('UPDATE dishes SET is_available = FALSE WHERE dish_id = ?', [req.params.id]);
    res.json({ message: '菜品已下架' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// ══════════════════════════════════════
// 分类管理
// ══════════════════════════════════════

// GET /api/admin/categories
router.get('/categories', async (req, res) => {
  try {
    const [rows] = await pool.query('SELECT * FROM categories ORDER BY sort_order');
    res.json(rows);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// POST /api/admin/categories
router.post('/categories', async (req, res) => {
  const { category_name, icon, description, sort_order } = req.body;
  if (!category_name) return res.status(400).json({ message: '分类名必填' });
  try {
    const [result] = await pool.query(
      'INSERT INTO categories (category_name, icon, description, sort_order) VALUES (?, ?, ?, ?)',
      [category_name, icon || '🍽️', description || '', sort_order || 0]
    );
    res.status(201).json({ message: '分类添加成功', category_id: result.insertId });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// PUT /api/admin/categories/:id
router.put('/categories/:id', async (req, res) => {
  const { category_name, icon, description, sort_order, is_active } = req.body;
  try {
    await pool.query(
      'UPDATE categories SET category_name=?, icon=?, description=?, sort_order=?, is_active=? WHERE category_id=?',
      [category_name, icon, description, sort_order, is_active !== false, req.params.id]
    );
    res.json({ message: '分类更新成功' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// DELETE /api/admin/categories/:id
router.delete('/categories/:id', async (req, res) => {
  try {
    await pool.query('UPDATE categories SET is_active = FALSE WHERE category_id = ?', [req.params.id]);
    res.json({ message: '分类已隐藏' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// ══════════════════════════════════════
// 订单管理
// ══════════════════════════════════════

// GET /api/admin/orders?status=pending
router.get('/orders', async (req, res) => {
  const { status } = req.query;
  let q = `
    SELECT o.*, t.table_number AS main_table_number,
           u.username, COUNT(oi.item_id) AS item_count
    FROM orders o
    JOIN \`tables\` t ON o.main_table_id = t.table_id
    LEFT JOIN users u ON o.user_id = u.user_id
    LEFT JOIN order_items oi ON o.order_id = oi.order_id
    WHERE 1=1
  `;
  const params = [];
  if (status) { q += ' AND o.status = ?'; params.push(status); }
  q += ' GROUP BY o.order_id ORDER BY o.created_at DESC';
  try {
    const [rows] = await pool.query(q, params);
    res.json(rows);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// GET /api/admin/orders/:id — 订单详情
router.get('/orders/:id', async (req, res) => {
  try {
    const [orders] = await pool.query(`
      SELECT o.*, t.table_number AS main_table_number
      FROM orders o JOIN \`tables\` t ON o.main_table_id = t.table_id
      WHERE o.order_id = ?
    `, [req.params.id]);
    if (!orders.length) return res.status(404).json({ message: '订单不存在' });

    const [items] = await pool.query(`
      SELECT oi.*, d.name, d.image_url
      FROM order_items oi JOIN dishes d ON oi.dish_id = d.dish_id
      WHERE oi.order_id = ?
    `, [req.params.id]);

    const [tables] = await pool.query(`
      SELECT t.table_number FROM order_tables ot
      JOIN \`tables\` t ON ot.table_id = t.table_id
      WHERE ot.order_id = ?
    `, [req.params.id]);

    res.json({ ...orders[0], items, tables });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// PUT /api/admin/orders/:id/status
router.put('/orders/:id/status', async (req, res) => {
  const { status } = req.body;
  const valid = ['pending','confirmed','completed','cancelled'];
  if (!valid.includes(status)) return res.status(400).json({ message: '无效状态' });
  try {
    await pool.query('UPDATE orders SET status = ? WHERE order_id = ?', [status, req.params.id]);
    // 若完成订单，同步菜品销量
    if (status === 'completed') {
      await pool.query(`
        UPDATE dishes d
        JOIN order_items oi ON d.dish_id = oi.dish_id
        SET d.sales_count = d.sales_count + oi.quantity
        WHERE oi.order_id = ?
      `, [req.params.id]);
    }
    res.json({ message: '状态更新成功' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// ══════════════════════════════════════
// 评论管理
// ══════════════════════════════════════

// GET /api/admin/comments?status=pending
router.get('/comments', async (req, res) => {
  const { status } = req.query;
  let q = `
    SELECT c.*, d.name AS dish_name
    FROM comments c JOIN dishes d ON c.dish_id = d.dish_id
    WHERE 1=1
  `;
  const params = [];
  if (status) { q += ' AND c.status = ?'; params.push(status); }
  q += ' ORDER BY c.created_at DESC LIMIT 100';
  try {
    const [rows] = await pool.query(q, params);
    res.json(rows);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// PUT /api/admin/comments/:id — 审核/拒绝
router.put('/comments/:id', async (req, res) => {
  const { status } = req.body;
  if (!['approved','rejected'].includes(status)) return res.status(400).json({ message: '无效状态' });
  try {
    await pool.query('UPDATE comments SET status = ? WHERE comment_id = ?', [status, req.params.id]);
    res.json({ message: '操作成功' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// DELETE /api/admin/comments/:id
router.delete('/comments/:id', async (req, res) => {
  try {
    await pool.query('DELETE FROM comments WHERE comment_id = ?', [req.params.id]);
    res.json({ message: '评论已删除' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// ══════════════════════════════════════
// 意见反馈管理
// ══════════════════════════════════════

// GET /api/admin/feedbacks
router.get('/feedbacks', async (req, res) => {
  const { status, category } = req.query;
  let q = 'SELECT * FROM feedbacks WHERE 1=1';
  const params = [];
  if (status)   { q += ' AND status = ?';   params.push(status); }
  if (category) { q += ' AND category = ?'; params.push(category); }
  q += ' ORDER BY created_at DESC';
  try {
    const [rows] = await pool.query(q, params);
    res.json(rows);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// PUT /api/admin/feedbacks/:id/status
router.put('/feedbacks/:id/status', async (req, res) => {
  const { status } = req.body;
  if (!['pending','read','resolved'].includes(status)) return res.status(400).json({ message: '无效状态' });
  try {
    await pool.query('UPDATE feedbacks SET status = ? WHERE feedback_id = ?', [status, req.params.id]);
    res.json({ message: '操作成功' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// ══════════════════════════════════════
// 用户管理
// ══════════════════════════════════════

// GET /api/admin/users
router.get('/users', async (req, res) => {
  try {
    const [rows] = await pool.query(`
      SELECT u.user_id, u.username, u.email, u.created_at,
             COUNT(o.order_id) AS order_count
      FROM users u
      LEFT JOIN orders o ON u.user_id = o.user_id
      GROUP BY u.user_id
      ORDER BY u.created_at DESC
    `);
    res.json(rows);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// ══════════════════════════════════════
// 统计数据（仪表盘）
// ══════════════════════════════════════

// GET /api/admin/stats
router.get('/stats', async (req, res) => {
  try {
    const [[{ total_orders }]]   = await pool.query("SELECT COUNT(*) AS total_orders FROM orders");
    const [[{ pending_orders }]] = await pool.query("SELECT COUNT(*) AS pending_orders FROM orders WHERE status='pending'");
    const [[{ total_revenue }]]  = await pool.query("SELECT IFNULL(SUM(total_price),0) AS total_revenue FROM orders WHERE status='completed'");
    const [[{ total_users }]]    = await pool.query("SELECT COUNT(*) AS total_users FROM users");
    const [[{ total_dishes }]]   = await pool.query("SELECT COUNT(*) AS total_dishes FROM dishes WHERE is_available=TRUE");
    const [[{ pending_comments }]] = await pool.query("SELECT COUNT(*) AS pending_comments FROM comments WHERE status='pending'");
    const [[{ pending_feedbacks }]] = await pool.query("SELECT COUNT(*) AS pending_feedbacks FROM feedbacks WHERE status='pending'");
    const [top_dishes] = await pool.query(
      'SELECT name, sales_count, average_rating FROM dishes WHERE is_available=TRUE ORDER BY sales_count DESC LIMIT 5'
    );
    res.json({ total_orders, pending_orders, total_revenue, total_users, total_dishes, pending_comments, pending_feedbacks, top_dishes });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// ══════════════════════════════════════
// 桌号 & 二维码管理
// ══════════════════════════════════════

// GET /api/admin/tables
router.get('/tables', async (req, res) => {
  try {
    const [rows] = await pool.query('SELECT * FROM `tables` ORDER BY table_number');
    res.json(rows);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// POST /api/admin/tables
router.post('/tables', async (req, res) => {
  const { table_number } = req.body;
  if (!table_number) return res.status(400).json({ message: '桌号必填' });
  try {
    const qrUrl = `${process.env.FRONTEND_URL || 'http://localhost:3000'}/order?table=${encodeURIComponent(table_number)}`;
    const [result] = await pool.query(
      'INSERT INTO `tables` (table_number, qr_code_url) VALUES (?, ?)',
      [table_number, qrUrl]
    );
    res.status(201).json({ message: '桌号添加成功', table_id: result.insertId, qr_code_url: qrUrl });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// DELETE /api/admin/tables/:id
router.delete('/tables/:id', async (req, res) => {
  try {
    await pool.query('DELETE FROM `tables` WHERE table_id = ?', [req.params.id]);
    res.json({ message: '桌号已删除' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

module.exports = router;
