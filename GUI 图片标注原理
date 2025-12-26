import sys
import json
import os
from PyQt6.QtWidgets import *
from PyQt6.QtCore import *
from PyQt6.QtGui import *

# --- 1. 全局配置与颜色映射 ---
TYPE_COLORS = {
    "按钮 (Button)": "#FF5722",      # 橙色
    "图标 (Icon)": "#2196F3",        # 蓝色
    "菜单项 (Menu Item)": "#9C27B0",  # 紫色
    "输入框 (Text Field)": "#4CAF50", # 绿色
    "下拉框 (Dropdown)": "#FFC107",   # 黄色
    "复选框/开关 (Checkbox)": "#00BCD4", # 青色
    "文本 (Text)": "#E91E63",        # 粉色
    "其他 (Other)": "#607D8B"        # 灰色
}
# 定义未选择类型时的默认颜色（醒目绿）
DEFAULT_UNSELECTED_COLOR = "#00FF00" 

# --- 2. 增强型标注框 ---
class DraggableRectItem(QGraphicsRectItem):
    def __init__(self, rect, label_id, parent_window, ui_type=""):
        super().__init__(rect)
        self.label_id = label_id
        self.parent_window = parent_window
        self.ui_type = ui_type
        self.handle_size = 8
        self.current_handle = 0
        
        self.setFlags(
            QGraphicsRectItem.GraphicsItemFlag.ItemIsMovable |
            QGraphicsRectItem.GraphicsItemFlag.ItemIsSelectable |
            QGraphicsRectItem.GraphicsItemFlag.ItemSendsGeometryChanges
        )
        self.setAcceptHoverEvents(True)

    def paint(self, painter, option, widget):
        # 如果没有类型，使用默认醒目颜色
        base_color = QColor(TYPE_COLORS.get(self.ui_type, DEFAULT_UNSELECTED_COLOR))
        is_selected = self.isSelected()
        
        pen = QPen(QColor("#FFFFFF") if is_selected else base_color, 2)
        if is_selected:
            pen.setStyle(Qt.PenStyle.DashLine)
        painter.setPen(pen)
        
        fill_color = QColor(base_color)
        fill_color.setAlpha(50 if not is_selected else 90)
        painter.setBrush(fill_color)
        
        painter.drawRoundedRect(self.rect(), 3, 3)

        if is_selected:
            center = self.rect().center()
            painter.setPen(QPen(Qt.GlobalColor.white, 2))
            painter.drawLine(QPointF(center.x() - 10, center.y()), QPointF(center.x() + 10, center.y()))
            painter.drawLine(QPointF(center.x(), center.y() - 10), QPointF(center.x(), center.y() + 10))
            painter.setPen(QPen(Qt.GlobalColor.red, 3))
            painter.drawPoint(center)

    def get_handle_at(self, pos):
        r, h = self.rect(), self.handle_size
        x, y = pos.x(), pos.y()
        if abs(x - r.left()) < h and abs(y - r.top()) < h: return 5
        if abs(x - r.right()) < h and abs(y - r.bottom()) < h: return 8
        return 0

    def mousePressEvent(self, event):
        self.current_handle = self.get_handle_at(event.pos())
        if self.current_handle != 0:
            self.setFlag(QGraphicsRectItem.GraphicsItemFlag.ItemIsMovable, False)
        else:
            self.setFlag(QGraphicsRectItem.GraphicsItemFlag.ItemIsMovable, True)
        self.parent_window.sync_list_selection(self.label_id)
        super().mousePressEvent(event)

    def mouseMoveEvent(self, event):
        if self.current_handle != 0:
            r = self.rect()
            if self.current_handle == 5: r.setTopLeft(event.pos())
            elif self.current_handle == 8: r.setBottomRight(event.pos())
            self.prepareGeometryChange()
            self.setRect(r.normalized())
        else:
            super().mouseMoveEvent(event)

    def mouseReleaseEvent(self, event):
        self.current_handle = 0
        super().mouseReleaseEvent(event)

# --- 3. 画布视图 ---
class AnnotationCanvas(QGraphicsView):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.scene = QGraphicsScene()
        self.setScene(self.scene)
        self.setRenderHint(QPainter.RenderHint.Antialiasing)
        self.setTransformationAnchor(QGraphicsView.ViewportAnchor.AnchorUnderMouse)
        self.setResizeAnchor(QGraphicsView.ViewportAnchor.AnchorUnderMouse)
        self.setBackgroundBrush(QColor("#1a1a1a"))
        
        self.image_item = None
        self.start_pos = None
        self.current_rect = None
        self.parent_window = parent
        self._is_panning = False
        self._pan_start = QPoint()

    def set_image(self, path):
        self.scene.clear()
        pixmap = QPixmap(path)
        if pixmap.isNull(): return
        self.image_item = self.scene.addPixmap(pixmap)
        self.scene.setSceneRect(QRectF(pixmap.rect()))
        self.fitInView(self.scene.sceneRect(), Qt.AspectRatioMode.KeepAspectRatio)

    def wheelEvent(self, event):
        factor = 1.15 if event.angleDelta().y() > 0 else 0.85
        self.scale(factor, factor)

    def mousePressEvent(self, event):
        if event.button() == Qt.MouseButton.MiddleButton:
            self._is_panning = True
            self._pan_start = event.pos()
            self.setCursor(Qt.CursorShape.ClosedHandCursor)
            return

        item = self.itemAt(event.pos())
        if isinstance(item, DraggableRectItem):
            super().mousePressEvent(event)
            return
        
        if self.image_item and event.button() == Qt.MouseButton.LeftButton:
            self.start_pos = self.mapToScene(event.pos())
            self.current_rect = self.scene.addRect(QRectF(self.start_pos, self.start_pos), QPen(QColor(DEFAULT_UNSELECTED_COLOR), 2))
        super().mousePressEvent(event)

    def mouseMoveEvent(self, event):
        if self._is_panning:
            delta = event.pos() - self._pan_start
            self._pan_start = event.pos()
            self.horizontalScrollBar().setValue(self.horizontalScrollBar().value() - delta.x())
            self.verticalScrollBar().setValue(self.verticalScrollBar().value() - delta.y())
            return

        if self.start_pos and self.current_rect:
            curr_pos = self.mapToScene(event.pos())
            self.current_rect.setRect(QRectF(self.start_pos, curr_pos).normalized())
        super().mouseMoveEvent(event)

    def mouseReleaseEvent(self, event):
        if event.button() == Qt.MouseButton.MiddleButton:
            self._is_panning = False
            self.setCursor(Qt.CursorShape.ArrowCursor)
            return

        if self.current_rect:
            rect = self.current_rect.rect()
            self.scene.removeItem(self.current_rect)
            if rect.width() > 5 and rect.height() > 5:
                new_rect = DraggableRectItem(rect, -1, self.parent_window)
                self.scene.addItem(new_rect)
                self.parent_window.add_new_label(new_rect)
            self.current_rect = None
            self.start_pos = None
        super().mouseReleaseEvent(event)

# --- 4. 数据结构 ---
class LabelItemData:
    def __init__(self, rect_item, label_id):
        self.label_id = label_id
        self.rect_item = rect_item
        self.type = "" 
        self.description = ""

# --- 5. 主窗口 ---
class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("WinUI 标注")
        self.resize(1400, 900)
        
        # 强制设置样式，解决文字看不清的问题
        self.setStyleSheet("""
            QMainWindow { background-color: #0d0d0d; }
            QWidget { color: #e0e0e0; font-family: 'Segoe UI', 'Microsoft YaHei'; }
            QFrame#SidePanel { background-color: #161616; border-right: 1px solid #333; }
            QLabel { color: #888; font-size: 12px; font-weight: bold; }
            QListWidget { background-color: #1e1e1e; border: 1px solid #333; border-radius: 6px; outline: none; }
            QListWidget::item { padding: 10px; border-bottom: 1px solid #252525; }
            QListWidget::item:selected { background-color: #2d2d2d; color: #00aaff; border-left: 3px solid #00aaff; }
            QPushButton { background-color: #333; border: none; border-radius: 4px; padding: 8px; color: white; }
            QPushButton:hover { background-color: #444; }
            QPushButton#SaveBtn { background-color: #0078d4; font-size: 14px; font-weight: bold; height: 40px; }
            QPushButton#SaveBtn:hover { background-color: #1085e0; }
            
            /* 下拉框样式修复 */
            QComboBox { 
                background-color: #252525; 
                border: 1px solid #3d3d3d; 
                border-radius: 4px; 
                padding: 5px 10px; 
                color: #e0e0e0; 
            }
            QComboBox:hover { border: 1px solid #00aaff; }
            /* 关键：修复下拉列表颜色，防止白底白字 */
            QComboBox QAbstractItemView { 
                background-color: #1e1e1e; 
                color: #e0e0e0;
                border: 1px solid #333; 
                selection-background-color: #2d2d2d; 
                selection-color: #00aaff; 
                outline: none; 
            }
            QComboBox::item { height: 30px; padding-left: 10px; color: #e0e0e0; }
            QComboBox::item:selected { background-color: #2d2d2d; color: #00aaff; }

            QTextEdit { background-color: #252525; border: 1px solid #333; border-radius: 4px; padding: 5px; color: #e0e0e0; }
        """)

        self.full_image_paths = []
        self.labels_data = []
        self.current_idx = -1
        self.init_ui()

    def init_ui(self):
        main_layout = QHBoxLayout()
        main_layout.setContentsMargins(0, 0, 0, 0)
        main_layout.setSpacing(0)

        # 侧边栏
        self.side_panel = QFrame()
        self.side_panel.setObjectName("SidePanel")
        self.side_panel.setFixedWidth(320)
        
        shadow = QGraphicsDropShadowEffect()
        shadow.setBlurRadius(20)
        shadow.setXOffset(5)
        shadow.setColor(QColor(0, 0, 0, 180))
        self.side_panel.setGraphicsEffect(shadow)

        side_layout = QVBoxLayout(self.side_panel)
        side_layout.setContentsMargins(20, 20, 20, 20)
        side_layout.setSpacing(12)

        title = QLabel("WINUI ANNOTATION")
        title.setStyleSheet("color: #00aaff; font-size: 16px; letter-spacing: 1px;")
        side_layout.addWidget(title)

        btn_layout = QHBoxLayout()
        self.btn_f = QPushButton("添加文件"); self.btn_d = QPushButton("添加文件夹")
        btn_layout.addWidget(self.btn_f); btn_layout.addWidget(self.btn_d)
        side_layout.addLayout(btn_layout)

        side_layout.addWidget(QLabel("图片队列 (右键可删除)"))
        self.file_list = QListWidget()
        self.file_list.setContextMenuPolicy(Qt.ContextMenuPolicy.CustomContextMenu)
        self.file_list.customContextMenuRequested.connect(self.show_file_context_menu)
        side_layout.addWidget(self.file_list)

        side_layout.addWidget(QLabel("已标注对象 (Del 删除)"))
        self.label_list = QListWidget()
        side_layout.addWidget(self.label_list)

        side_layout.addWidget(QLabel("UI 类型"))
        self.type_combo = QComboBox()
        self.type_combo.setView(QListView())
        self.type_combo.addItems(list(TYPE_COLORS.keys()))
        self.type_combo.setPlaceholderText("请选择类型...") 
        
        # --- 修复核心：初始化后立即重置为空 ---
        self.type_combo.setCurrentIndex(-1) 
        
        side_layout.addWidget(self.type_combo)

        side_layout.addWidget(QLabel("详细描述"))
        self.desc_edit = QTextEdit()
        self.desc_edit.setFixedHeight(80)
        side_layout.addWidget(self.desc_edit)

        self.btn_save = QPushButton("保存并继续 (Enter)")
        self.btn_save.setObjectName("SaveBtn")
        side_layout.addWidget(self.btn_save)

        self.canvas = AnnotationCanvas(self)
        main_layout.addWidget(self.side_panel)
        main_layout.addWidget(self.canvas)

        central_widget = QWidget()
        central_widget.setLayout(main_layout)
        self.setCentralWidget(central_widget)

        self.btn_f.clicked.connect(self.manual_add_files)
        self.btn_d.clicked.connect(self.manual_add_dir)
        self.btn_save.clicked.connect(self.save_and_next)
        self.type_combo.currentTextChanged.connect(self.update_type)
        self.desc_edit.textChanged.connect(self.update_desc)
        self.file_list.currentRowChanged.connect(self.switch_image)
        self.label_list.itemClicked.connect(self.handle_label_click)

    def show_file_context_menu(self, pos):
        item = self.file_list.itemAt(pos)
        if item:
            menu = QMenu(self)
            delete_action = QAction("从列表中移除此图片", self)
            delete_action.triggered.connect(lambda: self.delete_image_at_row(self.file_list.row(item)))
            menu.addAction(delete_action)
            menu.setStyleSheet("QMenu { background-color: #252525; color: white; } QMenu::item:selected { background-color: #0078d4; }")
            menu.exec(self.file_list.mapToGlobal(pos))

    def delete_image_at_row(self, row):
        if 0 <= row < len(self.full_image_paths):
            if row == self.current_idx:
                self.canvas.scene.clear()
                self.labels_data = []
                self.label_list.clear()
                self.desc_edit.clear()
                self.current_idx = -1
                # 删除当前图片后，重置UI
                self.type_combo.setCurrentIndex(-1)
            
            self.file_list.takeItem(row)
            self.full_image_paths.pop(row)
            
            if self.current_idx > row:
                self.current_idx -= 1
            
            if self.file_list.count() > 0 and self.current_idx == -1:
                self.file_list.setCurrentRow(0)

    # --- 核心逻辑 ---
    def add_new_label(self, item):
        lid = len(self.labels_data) + 1
        item.label_id = lid
        item.ui_type = "" 
        
        ld = LabelItemData(item, lid)
        ld.type = ""
        self.labels_data.append(ld)
        
        l_item = QListWidgetItem(f"#{lid} - [未选择]")
        l_item.setForeground(QColor(DEFAULT_UNSELECTED_COLOR))
        self.label_list.addItem(l_item)
        self.label_list.setCurrentRow(self.label_list.count()-1)

        # 重置 UI
        self.type_combo.blockSignals(True)
        self.type_combo.setCurrentIndex(-1) # 重置为空
        self.type_combo.blockSignals(False)
        
        self.desc_edit.blockSignals(True)
        self.desc_edit.clear()
        self.desc_edit.blockSignals(False)

        self.desc_edit.setFocus()

    def update_type(self, t):
        if not t: return # 防止空字符触发
        row = self.label_list.currentRow()
        if 0 <= row < len(self.labels_data):
            self.labels_data[row].type = t
            self.labels_data[row].rect_item.ui_type = t
            self.labels_data[row].rect_item.update() 
            item = self.label_list.item(row)
            item.setText(f"#{row+1} - {t}")
            item.setForeground(QColor(TYPE_COLORS.get(t, "#FFFFFF")))

    def update_desc(self):
        row = self.label_list.currentRow()
        if 0 <= row < len(self.labels_data):
            self.labels_data[row].description = self.desc_edit.toPlainText()

    def handle_label_click(self):
        row = self.label_list.currentRow()
        if 0 <= row < len(self.labels_data):
            d = self.labels_data[row]
            self.canvas.scene.clearSelection()
            d.rect_item.setSelected(True)
            
            self.type_combo.blockSignals(True)
            if d.type and d.type in TYPE_COLORS:
                self.type_combo.setCurrentText(d.type)
            else:
                self.type_combo.setCurrentIndex(-1)
            self.type_combo.blockSignals(False)
            
            self.desc_edit.blockSignals(True)
            self.desc_edit.setText(d.description)
            self.desc_edit.blockSignals(False)

    def sync_list_selection(self, lid):
        for i in range(self.label_list.count()):
            if f"#{lid}" in self.label_list.item(i).text():
                self.label_list.setCurrentRow(i)
                self.handle_label_click()
                break

    # --- 文件与保存 ---
    def manual_add_files(self):
        fs, _ = QFileDialog.getOpenFileNames(self, "选择图片", "", "Images (*.png *.jpg *.jpeg *.bmp)")
        if fs: self.handle_dropped_files(fs)

    def manual_add_dir(self):
        d = QFileDialog.getExistingDirectory(self, "选择文件夹")
        if d: self.handle_dropped_files([d])

    def handle_dropped_files(self, paths):
        exts = ('.png', '.jpg', '.jpeg', '.bmp')
        for p in paths:
            p = p.replace('\\', '/')
            if os.path.isdir(p):
                for f in os.listdir(p):
                    if f.lower().endswith(exts):
                        fp = f"{p}/{f}"
                        if fp not in self.full_image_paths:
                            self.full_image_paths.append(fp)
                            self.file_list.addItem(os.path.basename(fp))
            elif p.lower().endswith(exts):
                if p not in self.full_image_paths:
                    self.full_image_paths.append(p)
                    self.file_list.addItem(os.path.basename(p))
        if self.current_idx == -1 and self.full_image_paths:
            self.file_list.setCurrentRow(0)

    def switch_image(self, row):
        if row < 0 or row >= len(self.full_image_paths): return
        if self.current_idx != -1: self.save_data()
        self.current_idx = row
        self.canvas.set_image(self.full_image_paths[row])
        self.labels_data = []
        self.label_list.clear()
        
        json_p = os.path.splitext(self.full_image_paths[row])[0] + ".json"
        if os.path.exists(json_p):
            try:
                with open(json_p, 'r', encoding='utf-8') as f:
                    data = json.load(f)
                    for ann in data.get("annotations", []):
                        b = ann["bbox"]
                        t = ann.get("type", "")
                        item = DraggableRectItem(QRectF(b[0], b[1], b[2], b[3]), -1, self, t)
                        self.canvas.scene.addItem(item)
                        # 加载已有数据
                        lid = len(self.labels_data) + 1
                        item.label_id = lid
                        item.ui_type = t
                        ld = LabelItemData(item, lid)
                        ld.type = t
                        ld.description = ann.get("desc", "")
                        self.labels_data.append(ld)
                        
                        display_text = f"#{lid} - {t}" if t else f"#{lid} - [未选择]"
                        l_item = QListWidgetItem(display_text)
                        l_item.setForeground(QColor(TYPE_COLORS.get(t, DEFAULT_UNSELECTED_COLOR)))
                        self.label_list.addItem(l_item)
            except: pass

    def save_data(self):
        if self.current_idx == -1: return
        path = self.full_image_paths[self.current_idx]
        data = {"image": os.path.basename(path), "annotations": []}
        for l in self.labels_data:
            r = l.rect_item.rect()
            center = r.center()
            data["annotations"].append({
                "type": l.type, 
                "point": [int(center.x()), int(center.y())], 
                "bbox": [int(r.x()), int(r.y()), int(r.width()), int(r.height())],
                "desc": l.description
            })
        with open(os.path.splitext(path)[0]+".json", 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

    def save_and_next(self):
        self.save_data()
        if self.current_idx < len(self.full_image_paths) - 1:
            self.file_list.setCurrentRow(self.current_idx + 1)
        else:
            QMessageBox.information(self, "完成", "已经是最后一张图片了！")

    def keyPressEvent(self, event):
        if event.key() in (Qt.Key.Key_Return, Qt.Key.Key_Enter):
            self.save_and_next()
        elif event.key() == Qt.Key.Key_Delete:
            row = self.label_list.currentRow()
            if 0 <= row < len(self.labels_data):
                d = self.labels_data.pop(row)
                self.canvas.scene.removeItem(d.rect_item)
                self.label_list.takeItem(row)
                self.type_combo.setCurrentIndex(-1)
                self.desc_edit.clear()

if __name__ == "__main__":
    import ctypes # 必须导入这个库


    # 告诉 Windows 这是一个独立的应用程序，不要合并到 Python 进程图标中
    myappid = 'mycompany.myproduct.subproduct.version' # 随意写一个唯一的字符串
    ctypes.windll.shell32.SetCurrentProcessExplicitAppUserModelID(myappid)

    app = QApplication(sys.argv)
    
    # 在代码里也加载一下图标，确保窗口左上角显示
    # 假设你的图标文件名叫 logo.ico，放在代码同级目录下
    if os.path.exists("logo.ico"):
        app.setWindowIcon(QIcon("logo.ico"))
    
    app.setFont(QFont("Segoe UI", 10))
    w = MainWindow()
    w.show()
    sys.exit(app.exec())
    app = QApplication(sys.argv)
    app.setFont(QFont("Segoe UI", 10))
    w = MainWindow()
    w.show()
    sys.exit(app.exec())
