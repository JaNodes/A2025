import sys
import os
import subprocess
import threading
from PyQt5.QtWidgets import (
    QApplication, QWidget, QVBoxLayout, QHBoxLayout, QPushButton,
    QFileDialog, QLabel, QLineEdit, QListWidget, QListWidgetItem, QProgressBar,
    QSplitter, QScrollArea, QSizePolicy, QDialog, QMenu
)
from PyQt5.QtCore import Qt, QTimer, QEvent, pyqtSignal, QObject
from PyQt5.QtGui import QFont, QCursor, QIcon, QKeySequence, QPixmap, QPainter
from PyQt5.QtWidgets import QToolButton
from PyQt5.QtWidgets import QMessageBox
from PyQt5.QtWidgets import QTextEdit
from PyQt5.QtWidgets import QHBoxLayout
from PyQt5.QtWidgets import QInputDialog
import time
import traceback
import shutil

# Bạn cần đảm bảo rằng adb.exe và fastboot.exe nằm trong cùng thư mục với script
# ADB = "./adb.exe"
# FASTBOOT = "./fastboot.exe"
def get_resource_dir():
    # Khi chạy từ PyInstaller --onefile → tài nguyên nằm trong _MEIPASS
    if getattr(sys, 'frozen', False) and hasattr(sys, '_MEIPASS'):
        return sys._MEIPASS
    # Khi chạy dev (python .py)
    return os.path.dirname(os.path.abspath(__file__))

RESOURCE_DIR = get_resource_dir()
ADB = os.path.join(RESOURCE_DIR, "adb.exe")
FASTBOOT = os.path.join(RESOURCE_DIR, "fastboot.exe")

# ví dụ icon/ảnh:
ICON_DIR = os.path.join(RESOURCE_DIR, "icon")

# scrcpy có thể nằm trong thư mục con 'scrcpy'
SCRCPY_EXE = os.path.join(RESOURCE_DIR, "scrcpy", "scrcpy.exe")

def get_adb_devices():
    try:
        result = subprocess.run([ADB, "devices"], stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True,
                                timeout=10, creationflags=getattr(subprocess, "CREATE_NO_WINDOW", 0))
        return [line.split()[0] for line in result.stdout.splitlines() if "\tdevice" in line or "\tsideload" in line or "\trecovery" in line]
    except Exception as e:
        print("Get ADB devices error:", e)
        return []

def get_fastboot_devices():
    try:
        result = subprocess.run([FASTBOOT, "devices"], stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True,
                                timeout=10, creationflags=getattr(subprocess, "CREATE_NO_WINDOW", 0))
        output = result.stdout + result.stderr
        return [line.split()[0] for line in output.splitlines() if "fastboot" in line]
    except Exception as e:
        print("Get fastboot devices error:", e)
        return []

def run_with_timeout(cmd, timeout=10):
    """
    Chạy lệnh với timeout, trả về (result, error) — chạy ngầm, không mở CMD.
    """
    try:
        result = subprocess.run(
            cmd,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            timeout=timeout,
            encoding='utf-8',
            creationflags=getattr(subprocess, "CREATE_NO_WINDOW", 0)
        )
        return result, None
    except subprocess.TimeoutExpired:
        return None, f"Timeout >{timeout}s: {' '.join(cmd)}"
    except Exception as e:
        return None, f"Exception: {e}"


def get_device_model(serial):
    """
    Trả về tên model thân thiện.
    Ưu tiên:
      - ro.product.model
      - ro.product.device (cheeseburger/dumpling)
      - ro.build.product / ro.product.name / ro.vendor.product.device
      - /proc/cmdline (cheeseburger|dumpling) – phòng hờ
    """
    def _gp(prop):
        try:
            r = subprocess.run(
                [ADB, "-s", serial, "shell", "getprop", prop],
                stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True, timeout=3,
                creationflags=getattr(subprocess, "CREATE_NO_WINDOW", 0)
            )
            return (r.stdout or "").strip()
        except Exception:
            return ""

    # 1) Thử model thường
    model = _gp("ro.product.model")
    if model:
        if "A5000" in model or "OnePlus 5" in model:
            return "OnePlus 5"
        if "A5010" in model or "OnePlus 5T" in model:
            return "OnePlus 5T"
        return model

    # 2) Dựa theo device codename
    for key in [
        "ro.product.device",
        "ro.vendor.product.device",
        "ro.product.name",
        "ro.build.product",
    ]:
        dv = _gp(key).lower()
        if dv:
            if "cheeseburger" in dv:
                return "OnePlus 5"
            if "dumpling" in dv:
                return "OnePlus 5T"

    # 3) Phòng hờ đọc /proc/cmdline
    try:
        r = subprocess.run(
            [ADB, "-s", serial, "shell", "cat", "/proc/cmdline"],
            stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True, timeout=3,
            creationflags=getattr(subprocess, "CREATE_NO_WINDOW", 0)
        )
        cmdline = (r.stdout or "").lower()
        if "cheeseburger" in cmdline:
            return "OnePlus 5"
        if "dumpling" in cmdline:
            return "OnePlus 5T"
    except Exception:
        pass

    return "Unknown"

def run_silent(cmd, timeout=12000):
    try:
        env = os.environ.copy()
        env["PATH"] = RESOURCE_DIR + os.pathsep + os.path.join(RESOURCE_DIR, "scrcpy") + os.pathsep + env.get("PATH", "")
        result = subprocess.run(
            cmd,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            timeout=timeout,
            creationflags=getattr(subprocess, "CREATE_NO_WINDOW", 0),
            text=True,
            cwd=RESOURCE_DIR,
            env=env
        )
        if result.returncode != 0:
            print(f"Subprocess error: {' '.join(cmd)}\n{result.stderr.strip()}")
        return result
    except subprocess.TimeoutExpired:
        print("TIMEOUT:", " ".join(cmd))
        return None
    except Exception as e:
        print("Run silent error:", cmd, e)
        return None

class DeviceListWidget(QListWidget):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.setSelectionMode(QListWidget.ExtendedSelection)
        self.setStyleSheet("""
            QListWidget::item:selected {
                background: transparent;
                color: #2196F3;
                font-weight: bold;
            }
            QListWidget {
                selection-background-color: transparent;
            }
        """)

        # Load background.png từ thư mục icon cùng cấp với adb.exe
        tool_dir = os.path.dirname(os.path.abspath(__file__))
        bg_path = os.path.join(ICON_DIR, "background.png")
        if os.path.exists(bg_path):
            self.bg_pixmap = QPixmap(bg_path)
        else:
            self.bg_pixmap = None

    def paintEvent(self, event):
        super().paintEvent(event)
        if hasattr(self, 'bg_pixmap') and self.bg_pixmap:
            painter = QPainter(self.viewport())
            painter.setOpacity(0.18)  # Độ mờ cho ảnh nền
            painter.drawPixmap(self.rect(), self.bg_pixmap)
            painter.end()

    def mousePressEvent(self, event):
        if event.button() == Qt.LeftButton:
            item = self.itemAt(event.pos())
            if item:
                idx = self.row(item)
                sel = self.selectionModel().isSelected(self.model().index(idx, 0))
                modifiers = QApplication.keyboardModifiers()
                if modifiers & Qt.ControlModifier:
                    if sel:
                        self.selectionModel().select(self.model().index(idx, 0), self.selectionModel().Deselect)
                    else:
                        self.selectionModel().select(self.model().index(idx, 0), self.selectionModel().Select)
                else:
                    if sel:
                        self.selectionModel().select(self.model().index(idx, 0), self.selectionModel().Deselect)
                    else:
                        super().mousePressEvent(event)
            else:
                super().mousePressEvent(event)
        else:
            super().mousePressEvent(event)

    def keyPressEvent(self, event):
        if event.modifiers() & Qt.ControlModifier and event.key() == Qt.Key_A:
            self.selectAll()
            event.accept()
            return
        if event.modifiers() & Qt.ControlModifier and event.key() == Qt.Key_D:
            self.clearSelection()
            event.accept()
            return
        super().keyPressEvent(event)

class PushProgressSignal(QObject):
    update = pyqtSignal(str, int)
    finished = pyqtSignal()

class CopyProgressDialog(QDialog):
    def __init__(self, serials, parent=None):
        super().__init__(parent)
        self.setWindowTitle("Tiến độ copy file trên từng thiết bị")
        self.setMinimumWidth(500)
        self.layout = QVBoxLayout(self)
        self.bars = {}
        self.labels = {}
        for serial in serials:
            row = QHBoxLayout()
            label = QLabel(serial)
            label.setFixedWidth(200)
            bar = QProgressBar()
            bar.setValue(0)
            bar.setMinimum(0)
            bar.setMaximum(100)
            bar.setFormat('%p%')
            bar.setFixedHeight(22)
            row.addWidget(label)
            row.addWidget(bar)
            self.layout.addLayout(row)
            self.bars[serial] = bar
            self.labels[serial] = label
        self.setLayout(self.layout)

    def set_percent(self, serial, percent):
        if serial in self.bars:
            self.bars[serial].setValue(int(percent))

class AndroidMultiTool(QWidget):

    def open_multi_screen_window(self):
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            QMessageBox.warning(self, "Thông báo", "Vui lòng chọn thiết bị muốn mở màn hình!")
            return

        # Chỉ cho phép mở scrcpy với thiết bị đang ở chế độ ADB
        adb_devices = get_adb_devices()
        valid_devices = [s for s in selected_devices if s in adb_devices]
        if not valid_devices:
            QMessageBox.warning(self, "Thông báo",
                                "Chỉ có thể mở màn hình cho thiết bị đang ở chế độ ADB!\n\nVui lòng chọn lại thiết bị.")
            return
        if len(valid_devices) < len(selected_devices):
            QMessageBox.information(self, "Lưu ý",
                                    "Một số thiết bị bạn chọn không ở chế độ ADB nên sẽ bị bỏ qua.")
        selected_devices = valid_devices

        # === TÌM scrcpy.exe THEO THƯ MỤC TÀI NGUYÊN (_MEIPASS) ===
        candidates = [
            os.path.join(RESOURCE_DIR, "scrcpy", "scrcpy.exe"),
            os.path.join(RESOURCE_DIR, "scrcpy.exe"),
        ]
        # Cho phép fallback khi chạy dev: scrcpy trong PATH (optional)
        path_scrcpy = shutil.which("scrcpy")
        if path_scrcpy:
            candidates.append(path_scrcpy)

        scrcpy_path = next((p for p in candidates if p and os.path.isfile(p)), None)

        if not scrcpy_path:
            QMessageBox.warning(
                self, "Thiếu scrcpy",
                "Không tìm thấy scrcpy.exe trong tài nguyên.\n"
                "Hãy đóng gói kèm thư mục scrcpy hoặc cài scrcpy trong PATH khi chạy dev."
            )
            return

        # Chuẩn bị env/cwd để scrcpy nhìn thấy DLL đi kèm
        env = os.environ.copy()
        env["PATH"] = os.pathsep.join([
            os.path.join(RESOURCE_DIR, "scrcpy"),
            RESOURCE_DIR,
            env.get("PATH", "")
        ])
        creationflags = getattr(subprocess, "CREATE_NO_WINDOW", 0) if os.name == "nt" else 0

        def open_screen(serial):
            try:
                subprocess.Popen([scrcpy_path, "--serial", serial],
                                creationflags=creationflags,
                                cwd=RESOURCE_DIR,
                                env=env)
            except Exception as e:
                QMessageBox.warning(self, "Lỗi", f"Không thể mở màn hình cho {serial}: {e}")

        for idx, serial in enumerate(selected_devices):
            threading.Timer(idx * 0.5, open_screen, args=(serial,)).start()


    def twrp_wipe_advanced(self):
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status("Đang thực hiện Wipe nâng cao (TWRP) cho các thiết bị...")
                n = len(selected_devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = n * 3  # 3 bước cho mỗi thiết bị

                def wipe_all(serial):
                    # 1. Wipe Advanced
                    try:
                        cmd = [ADB, "-s", serial, "shell", "twrp wipe dalvik"]
                        result = run_silent(cmd)
                        if result and result.returncode == 0:
                            self.log_device_result(serial, "Đã wipe Dalvik (TWRP)", status="OK")
                        else:
                            err = result.stderr.strip() if result else "Không rõ"
                            self.log_device_result(serial, f"Lỗi wipe Dalvik - {err}", status="ERROR")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi wipe Dalvik: {e}", status="ERROR")
                    finally:
                        with self.progress_lock:
                            self.done += 1

                    # 2. Erase Cache and Data
                    try:
                        cmd = [ADB, "-s", serial, "shell", "twrp wipe cache"]
                        result = run_silent(cmd)
                        if result and result.returncode == 0:
                            self.log_device_result(serial, "Đã wipe Cache (TWRP)", status="OK")
                        else:
                            err = result.stderr.strip() if result else "Không rõ"
                            self.log_device_result(serial, f"Lỗi wipe Cache - {err}", status="ERROR")
                        cmd2 = [ADB, "-s", serial, "shell", "twrp wipe data"]
                        result2 = run_silent(cmd2)
                        if result2 and result2.returncode == 0:
                            self.log_device_result(serial, "Đã wipe Data (TWRP)", status="OK")
                        else:
                            err2 = result2.stderr.strip() if result2 else "Không rõ"
                            self.log_device_result(serial, f"Lỗi wipe Data - {err2}", status="ERROR")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi wipe Cache/Data: {e}", status="ERROR")
                    finally:
                        with self.progress_lock:
                            self.done += 1

                    # 3. Format Data
                    try:
                        cmd = [ADB, "-s", serial, "shell", "twrp format data"]
                        result = run_silent(cmd)
                        if result and result.returncode == 0:
                            self.log_device_result(serial, "Đã format Data (TWRP)", status="OK")
                        else:
                            err = result.stderr.strip() if result else "Không rõ"
                            self.log_device_result(serial, f"Lỗi format Data - {err}", status="ERROR")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi format Data: {e}", status="ERROR")
                    finally:
                        with self.progress_lock:
                            self.done += 1

                    # 4. Nếu là OnePlus 5/5T, xóa sạch bằng lệnh đặc biệt
                    model = get_device_model(serial)
                    if model in ["OnePlus 5", "OnePlus 5T"]:
                        try:
                            cmds = [
                                [ADB, "-s", serial, "shell", "umount /data || true"],
                                [ADB, "-s", serial, "shell", "umount /cache || true"],
                                [ADB, "-s", serial, "shell", "rm -rf /data/* /data/.??*"],
                                [ADB, "-s", serial, "shell", "rm -rf /cache/* /cache/.??*"]
                            ]
                            for cmdx in cmds:
                                run_silent(cmdx)
                            self.log_device_result(serial, "Đã xóa sạch dữ liệu (OnePlus 5/5T)", status="OK")
                        except Exception as e:
                            self.log_device_result(serial, f"Lỗi xóa sạch dữ liệu: {e}", status="ERROR")
                        finally:
                            with self.progress_lock:
                                self.done += 1

                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=wipe_all, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã thực hiện Wipe nâng cao cho {n} thiết bị.")
                with self.progress_lock:
                    self.done = n * 3
                    self.total = n * 3
            except Exception as e:
                print("Lỗi tổng thể trong twrp_wipe_advanced:", e)
                self.set_status("Có lỗi trong thao tác Wipe nâng cao!")
        self.run_threaded(work)
    log_signal = pyqtSignal(str, str, str)  # serial, message, status
    
    def run_device_action(self, devices, action, status_start, status_done, total=None):
        if not devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status(status_start)
                n = len(devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = total if total else n
                threads = []
                for serial in devices:
                    t = threading.Thread(target=action, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(status_done.format(n=n))
                with self.progress_lock:
                    self.done = self.total
            except Exception as e:
                print(f"Lỗi tổng thể: {e}")
                self.set_status("Có lỗi trong thao tác!")
        self.run_threaded(work)

    def __init__(self):
        super().__init__()
        self.log_signal.connect(self._append_log)
        self.setWindowIcon(QIcon(os.path.join(ICON_DIR, "OKPay.ico")))
        self.setWindowTitle("Android Multi Tool (ADB/Fastboot)")
        self.setGeometry(400, 120, 1200, 700)

        # ====== KHỐI BÊN TRÁI: PHÍM CHỨC NĂNG ======
        self.left_widget = QWidget()
        self.left_layout = QVBoxLayout(self.left_widget)
        icon_path = os.path.join("icon", "file_choose.ico")
        icon_choose = QIcon(icon_path) if os.path.exists(icon_path) else QIcon()
        self.label_I = QLabel("I. Chế độ sử dụng")
        self.label_I.setFont(QFont("Arial", weight=QFont.Bold))
        self.left_layout.addWidget(self.label_I)
        row_reboot = QHBoxLayout()
        self.btn_reboot_all = QPushButton("Khởi động lại máy")
        self.btn_reboot_all.clicked.connect(self.adb_reboot_selected)
        row_reboot.addWidget(self.btn_reboot_all)
        self.btn_adb_reboot_recovery = QPushButton("Vào Recovery")
        self.btn_adb_reboot_recovery.clicked.connect(self.adb_reboot_recovery_selected)
        row_reboot.addWidget(self.btn_adb_reboot_recovery)
        self.left_layout.addLayout(row_reboot)
        self.left_layout.addSpacing(9)
        self.label_II = QLabel("II. Chế độ Fastboot")
        self.label_II.setFont(QFont("Arial", weight=QFont.Bold))
        self.left_layout.addWidget(self.label_II)
        row_fastboot = QHBoxLayout()
        self.btn_reboot_fastboot = QPushButton("Vào Fastboot")
        self.btn_reboot_fastboot.clicked.connect(self.adb_reboot_fastboot)
        row_fastboot.addWidget(self.btn_reboot_fastboot)

        self.btn_fastboot_reboot = QPushButton("Khởi động lại máy")
        self.btn_fastboot_reboot.clicked.connect(self.fastboot_reboot_selected)
        row_fastboot.addWidget(self.btn_fastboot_reboot)
        self.btn_fastboot_reboot_recovery = QPushButton("Vào Recovery")
        self.btn_fastboot_reboot_recovery.clicked.connect(self.fastboot_reboot_recovery_selected)
        row_fastboot.addWidget(self.btn_fastboot_reboot_recovery)

        self.left_layout.addLayout(row_fastboot)
        self.left_layout.addSpacing(9)
        self.label_III = QLabel("III. Chạy lệnh tuỳ chọn")
        self.label_III.setFont(QFont("Arial", weight=QFont.Bold))
        self.left_layout.addWidget(self.label_III)
        self.label_fastboot_cmd = QLabel("<b>Nhập lệnh Fastboot (như cmd, có thể dùng {serial}):</b>")
        self.left_layout.addWidget(self.label_fastboot_cmd)
        self.line_fastboot_cmd = QLineEdit()
        self.left_layout.addWidget(self.line_fastboot_cmd)
        self.btn_run_fastboot_cmd = QPushButton("Run command")
        self.btn_run_fastboot_cmd.clicked.connect(self.run_fastboot_command_selected)
        self.left_layout.addWidget(self.btn_run_fastboot_cmd)
        self.left_layout.addSpacing(9)
        self.label_IV = QLabel("IV. Cài đặt APK cho thiết bị")
        self.label_IV.setFont(QFont("Arial", weight=QFont.Bold))
        self.left_layout.addWidget(self.label_IV)
        self.label_apk = QLabel("<b>Chọn file APK để cài đặt (có thể chọn nhiều):</b>")
        self.left_layout.addWidget(self.label_apk)
        apk_row = QHBoxLayout()
        self.line_apk = QLineEdit()
        apk_row.addWidget(self.line_apk)
        self.btn_browse_apk = QPushButton("Chọn file APK")
        self.btn_browse_apk.setIcon(icon_choose)
        self.btn_browse_apk.clicked.connect(self.browse_apk)
        apk_row.addWidget(self.btn_browse_apk)
        self.left_layout.addLayout(apk_row)
        self.btn_install_apk = QPushButton("Bắt đầu cài APK")
        self.btn_install_apk.clicked.connect(self.adb_install_apk_selected)
        self.left_layout.addWidget(self.btn_install_apk)
        self.left_layout.addSpacing(9)
        self.label_V = QLabel("V. Boot file .img bằng Fastboot")
        self.label_V.setFont(QFont("Arial", weight=QFont.Bold))
        self.left_layout.addWidget(self.label_V)
        self.label_img = QLabel("<b>Chọn file .img để boot bằng Fastboot:</b>")
        self.left_layout.addWidget(self.label_img)
        img_row = QHBoxLayout()
        self.line_img = QLineEdit()
        img_row.addWidget(self.line_img)
        self.btn_browse_img = QPushButton("Chọn file .img")
        self.btn_browse_img.setIcon(icon_choose)
        self.btn_browse_img.clicked.connect(self.browse_img)
        img_row.addWidget(self.btn_browse_img)
        self.left_layout.addLayout(img_row)
        self.btn_fastboot_boot = QPushButton("Run Fastboot Boot")
        self.btn_fastboot_boot.clicked.connect(self.fastboot_boot_selected)
        self.left_layout.addWidget(self.btn_fastboot_boot)
        self.left_layout.addSpacing(9)
        self.label_VI = QLabel("VI. Cài ROM qua ADB Sideload")
        self.label_VI.setFont(QFont("Arial", weight=QFont.Bold))
        self.left_layout.addWidget(self.label_VI)
        self.label_zip = QLabel("<b>Chọn file .zip để sideload:</b>")
        self.left_layout.addWidget(self.label_zip)
        zip_row = QHBoxLayout()
        self.line_zip = QLineEdit()
        zip_row.addWidget(self.line_zip)
        self.btn_browse_zip = QPushButton("Chọn file ZIP")
        self.btn_browse_zip.setIcon(icon_choose)
        self.btn_browse_zip.clicked.connect(self.browse_zip)
        zip_row.addWidget(self.btn_browse_zip)
        self.left_layout.addLayout(zip_row)
        self.btn_adb_sideload = QPushButton("Run ADB Sideload")
        self.btn_adb_sideload.clicked.connect(self.adb_sideload_selected)
        self.left_layout.addWidget(self.btn_adb_sideload)
        self.left_layout.addSpacing(9)
        self.label_VII = QLabel("VII. Copy file sang thiết bị")
        self.label_VII.setFont(QFont("Arial", weight=QFont.Bold))
        self.left_layout.addWidget(self.label_VII)
        self.label_push = QLabel("<b>Chọn file để copy vào thiết bị (/sdcard/, có thể chọn nhiều):</b>")
        self.left_layout.addWidget(self.label_push)
        push_row = QHBoxLayout()
        self.line_push = QLineEdit()
        push_row.addWidget(self.line_push)
        self.btn_browse_push = QPushButton("Chọn file để copy")
        self.btn_browse_push.setIcon(icon_choose)
        self.btn_browse_push.clicked.connect(self.browse_push)
        push_row.addWidget(self.btn_browse_push)
        self.left_layout.addLayout(push_row)
        self.btn_push_file = QPushButton("Run Copy File")
        self.btn_push_file.clicked.connect(self.adb_push_selected)
        self.left_layout.addWidget(self.btn_push_file)
        self.left_layout.addSpacing(9)
        self.label_VIII = QLabel("VIII. Cài ZIP (ROM, patch) qua TWRP")
        self.label_VIII.setFont(QFont("Arial", weight=QFont.Bold))
        self.left_layout.addWidget(self.label_VIII)
        self.label_install_zip = QLabel("<b>Nhập tên file .zip đã có sẵn trên điện thoại để cài đặt qua TWRP:</b>")
        self.left_layout.addWidget(self.label_install_zip)
        self.line_install_zip = QLineEdit()
        self.left_layout.addWidget(self.line_install_zip)
        self.btn_install_zip_twrp = QPushButton("Cài ZIP (TWRP)")
        self.btn_install_zip_twrp.clicked.connect(self.twrp_install_zip)
        # Thêm nút cài APK qua TWRP bên cạnh nút cài ZIP
        row_twrp_install = QHBoxLayout()
        row_twrp_install.addWidget(self.btn_install_zip_twrp)
        self.btn_install_apk_twrp = QPushButton("Cài APK (TWRP)")
        self.btn_install_apk_twrp.clicked.connect(self.twrp_install_apk)
        row_twrp_install.addWidget(self.btn_install_apk_twrp)
        self.left_layout.addLayout(row_twrp_install)
        self.left_layout.addSpacing(9)
        self.label_IX = QLabel("IX. Wipe nâng cao (TWRP)")
        self.label_IX.setFont(QFont("Arial", weight=QFont.Bold))
        self.left_layout.addWidget(self.label_IX)
        self.btn_wipe_advanced = QPushButton("Wipe Advanced (TWRP) - Xóa Dalvik, Cache, System, Vendor, Data, Internal")
        self.btn_wipe_advanced.clicked.connect(self.twrp_wipe_advanced)
        self.left_layout.addWidget(self.btn_wipe_advanced)
        self.btn_wipe_cache_data = QPushButton("Erase Cache and Data (TWRP)")
        self.btn_wipe_cache_data.clicked.connect(self.twrp_wipe_cache_data)
        self.left_layout.addWidget(self.btn_wipe_cache_data)
        self.btn_wipe_data = QPushButton("Format Data (TWRP)")
        self.btn_wipe_data.clicked.connect(self.twrp_wipe_data)
        self.left_layout.addWidget(self.btn_wipe_data)
        self.left_layout.addSpacing(9)
        self.label_X = QLabel("X. Khởi động lại/chuyển chế độ trong TWRP")
        self.label_X.setFont(QFont("Arial", weight=QFont.Bold))
        self.left_layout.addWidget(self.label_X)
        self.btn_twrp_reboot_fastboot = QPushButton("Go to Fastboot (TWRP)")
        self.btn_twrp_reboot_fastboot.clicked.connect(self.twrp_reboot_fastboot)
        self.left_layout.addWidget(self.btn_twrp_reboot_fastboot)
        self.btn_twrp_reboot_recovery = QPushButton("Go to Recovery (TWRP)")
        self.btn_twrp_reboot_recovery.clicked.connect(self.twrp_reboot_recovery)
        self.left_layout.addWidget(self.btn_twrp_reboot_recovery)
        self.btn_twrp_reboot = QPushButton("Reboot")
        self.btn_twrp_reboot.clicked.connect(self.twrp_reboot)
        self.left_layout.addWidget(self.btn_twrp_reboot)
        # ====== XI. MultiCommand ROM Oxygen to ROM Tool ======
        self.label_XI = QLabel("XI. MultiCommand ROM OxygenOS to ROM Tool")
        self.label_XI.setFont(QFont("Arial", weight=QFont.Bold))
        self.left_layout.addWidget(self.label_XI)

        # Dòng chọn .img
        row_xi = QHBoxLayout()
        self.line_img_xi = QLineEdit()
        self.line_img_xi.setPlaceholderText("Chọn file .img để boot")
        row_xi.addWidget(self.line_img_xi)
        self.btn_browse_img_xi = QPushButton("Chọn .img")
        self.btn_browse_img_xi.clicked.connect(self.browse_img_xi)
        row_xi.addWidget(self.btn_browse_img_xi)
        self.left_layout.addLayout(row_xi)

        # Dòng chọn .zip cho 5
        row_zip5_xi = QHBoxLayout()
        self.line_zip5_xi = QLineEdit()
        self.line_zip5_xi.setPlaceholderText("Chọn file .zip cho 5")
        row_zip5_xi.addWidget(self.line_zip5_xi)
        self.btn_browse_zip5_xi = QPushButton("Chọn .zip cho 5")
        self.btn_browse_zip5_xi.clicked.connect(self.browse_zip5_xi)
        row_zip5_xi.addWidget(self.btn_browse_zip5_xi)
        self.left_layout.addLayout(row_zip5_xi)

        # Dòng chọn .zip cho 5T
        row_zip5t_xi = QHBoxLayout()
        self.line_zip5t_xi = QLineEdit()
        self.line_zip5t_xi.setPlaceholderText("Chọn file .zip cho 5T")
        row_zip5t_xi.addWidget(self.line_zip5t_xi)
        self.btn_browse_zip5t_xi = QPushButton("Chọn .zip cho 5T")
        self.btn_browse_zip5t_xi.clicked.connect(self.browse_zip5t_xi)
        row_zip5t_xi.addWidget(self.btn_browse_zip5t_xi)
        self.left_layout.addLayout(row_zip5t_xi)

        self.btn_start_xi = QPushButton("Start")
        self.btn_start_xi.clicked.connect(self.multi_command_rom_oxygen_to_rom_tool)
        self.left_layout.addWidget(self.btn_start_xi)

        # ====== XII. MultiCommand ROM Tool to Other ROM ======
        self.label_XII = QLabel("XII. MultiCommand ROM Tool to Other ROM")
        self.label_XII.setFont(QFont("Arial", weight=QFont.Bold))
        self.left_layout.addWidget(self.label_XII)

        # Dòng chọn .img
        row_xii_img = QHBoxLayout()
        self.line_img_xii = QLineEdit()
        self.line_img_xii.setPlaceholderText("Chọn file .img để boot")
        row_xii_img.addWidget(self.line_img_xii)
        self.btn_browse_img_xii = QPushButton("Chọn .img")
        self.btn_browse_img_xii.clicked.connect(self.browse_img_xii)
        row_xii_img.addWidget(self.btn_browse_img_xii)
        self.left_layout.addLayout(row_xii_img)

        # Dòng chọn .zip cho 5
        row_zip5_xii = QHBoxLayout()
        self.line_zip5_xii = QLineEdit()
        self.line_zip5_xii.setPlaceholderText("Chọn file .zip cho 5")
        row_zip5_xii.addWidget(self.line_zip5_xii)
        self.btn_browse_zip5_xii = QPushButton("Chọn .zip cho 5")
        self.btn_browse_zip5_xii.clicked.connect(self.browse_zip5_xii)
        row_zip5_xii.addWidget(self.btn_browse_zip5_xii)
        self.left_layout.addLayout(row_zip5_xii)

        # Dòng chọn .zip cho 5T
        row_zip5t_xii = QHBoxLayout()
        self.line_zip5t_xii = QLineEdit()
        self.line_zip5t_xii.setPlaceholderText("Chọn file .zip cho 5T")
        row_zip5t_xii.addWidget(self.line_zip5t_xii)
        self.btn_browse_zip5t_xii = QPushButton("Chọn .zip cho 5T")
        self.btn_browse_zip5t_xii.clicked.connect(self.browse_zip5t_xii)
        row_zip5t_xii.addWidget(self.btn_browse_zip5t_xii)
        self.left_layout.addLayout(row_zip5t_xii)

        self.btn_start_xii = QPushButton("Start")
        self.btn_start_xii.clicked.connect(self.multi_command_rom_tool_to_other_rom)
        self.left_layout.addWidget(self.btn_start_xii)

        self.left_layout.addStretch(1)
        #self.left_widget.setFixedWidth(550)  # Đặt độ rộng cố định cho khối chức năng
        self.scroll_area = QScrollArea()
        self.scroll_area.setWidgetResizable(True)
        self.scroll_area.setWidget(self.left_widget)
        self.scroll_area.setMinimumWidth(450)
        self.scroll_area.setMaximumWidth(500)

        # ====== KHỐI BÊN PHẢI: DANH SÁCH THIẾT BỊ, LOG, STATUS ======
        self.right_layout = QVBoxLayout()
        self.right_panel = QWidget()
        self.right_panel.setLayout(self.right_layout)
        btn_layout = QHBoxLayout()
        self.btn_scan = QPushButton("Quét thiết bị")
        self.btn_scan.clicked.connect(self.update_device_list)
        btn_layout.addWidget(self.btn_scan)
        # Gộp các nút chọn thiết bị thành 1 nút với menu
        self.btn_select_device = QPushButton("Chọn thiết bị")
        self.btn_select_device.setFixedWidth(140)
        self.select_device_menu = QMenu(self.btn_select_device)
        self.action_select_all = self.select_device_menu.addAction("Chọn tất cả")
        self.action_select_fastboot = self.select_device_menu.addAction("Chọn Fastboot")
        self.action_select_user = self.select_device_menu.addAction("Chọn thiết bị ở chế độ User")
        self.action_select_twrp = self.select_device_menu.addAction("Chọn thiết bị ở chế độ TWRP")
        self.action_select_sideload = self.select_device_menu.addAction("Chọn thiết bị ở chế độ Sideload")
        self.btn_select_device.setMenu(self.select_device_menu)
        btn_layout.addWidget(self.btn_select_device)
        self.btn_deselect_all = QPushButton("Bỏ chọn tất cả")
        self.btn_deselect_all.clicked.connect(self.deselect_all_devices)
        btn_layout.addWidget(self.btn_deselect_all)
        # Thêm nút Mở màn hình cạnh nút chọn thiết bị
        self.btn_open_screen = QPushButton("Mở màn hình")
        self.btn_open_screen.setFixedWidth(110)
        self.btn_open_screen.clicked.connect(self.open_multi_screen_window)
        btn_layout.addWidget(self.btn_open_screen)
        btn_layout.addStretch()

        # Kết nối các action menu
        self.action_select_all.triggered.connect(self.select_all_devices)
        self.action_select_fastboot.triggered.connect(self.select_fastboot_devices)
        self.action_select_user.triggered.connect(self.select_user_devices)
        self.action_select_twrp.triggered.connect(self.select_twrp_devices)
        self.action_select_sideload.triggered.connect(self.select_sideload_devices)

        self.search_box_area = QWidget()
        self.search_box_layout = QHBoxLayout(self.search_box_area)
        self.search_box = QLineEdit()
        self.search_box.setPlaceholderText("Tìm chức năng...")
        self.search_box.setFixedWidth(200)
        self.search_box.textChanged.connect(self.update_search_results)
        self.search_box.returnPressed.connect(self.goto_next_search)
        self.search_count_label = QLabel("")
        self.btn_next_search = QPushButton("Next")
        self.btn_next_search.setFixedWidth(60)
        self.btn_next_search.clicked.connect(self.goto_next_search)
        self.btn_hide_search = QPushButton("✕")
        self.btn_hide_search.setFixedWidth(25)
        self.btn_hide_search.clicked.connect(self.hide_search_box)
        self.search_box_layout.addWidget(self.search_box)
        self.search_box_layout.addWidget(self.search_count_label)
        self.search_box_layout.addWidget(self.btn_next_search)
        self.search_box_layout.addWidget(self.btn_hide_search)
        self.search_box_layout.setContentsMargins(0, 0, 0, 0)
        self.search_box_area.setLayout(self.search_box_layout)
        self.search_box_area.setVisible(False)
        btn_layout.addWidget(self.search_box_area)
        self.btn_toggle_guide = QPushButton("Hướng dẫn >>")
        self.btn_toggle_guide.clicked.connect(self.toggle_guide)
        btn_layout.addWidget(self.btn_toggle_guide)
        self.right_layout.addLayout(btn_layout)
        self.device_label = QLabel("Danh sách thiết bị (0/0):")
        self.right_layout.addWidget(self.device_label)
        self.device_list = DeviceListWidget()
        self.device_list.setMinimumHeight(500)
        self.device_list.itemSelectionChanged.connect(self.update_selected_count)
        self.right_layout.addWidget(self.device_label)
        self.right_layout.addWidget(self.device_list)

        # --- [1] Khung log QTextEdit ---
        log_header_layout = QHBoxLayout()
        log_label = QLabel("Log chi tiết từng thiết bị:")
        btn_clear_log = QPushButton("Xóa log")
        btn_clear_log.setFixedWidth(70)
        log_header_layout.addWidget(log_label)
        log_header_layout.addStretch()
        log_header_layout.addWidget(btn_clear_log)
        self.right_layout.addLayout(log_header_layout)

        self.device_log = QTextEdit()
        self.device_log.setReadOnly(True)
        self.device_log.setMinimumHeight(300)
        self.device_log.setStyleSheet("""
            QTextEdit {
                background: #23272e;
                color: #e2f5fc;
                font: 9pt Consolas, Courier;
                border-radius: 7px;
                border: 1px solid #789;
                padding: 6px;
            }
        """)
        self.right_layout.addWidget(self.device_log, stretch=1)
        btn_clear_log.clicked.connect(self.device_log.clear)
        # --- [1] END ---

        self.right_layout.addStretch(1)
        self.status_progress_layout = QHBoxLayout()

        self.status_label = QLabel("Sẵn sàng thao tác.")
        self.status_label.setStyleSheet("font-weight:bold;color:#388e3c;")
        self.status_label.setMinimumWidth(360)
        self.progress_bar = QProgressBar()
        self.progress_bar.setMinimum(0)
        self.progress_bar.setMaximum(100)
        self.progress_bar.setValue(0)
        self.progress_bar.setFormat('%p%')
        self.progress_bar.setTextVisible(True)
        self.progress_bar.setFixedWidth(230)
        self.progress_bar.setStyleSheet("""
            QProgressBar {
                border: 1px solid #bbb;
                border-radius: 7px;
                background: #fff;
                height: 24px;
                text-align: center;
                font: bold 12pt Arial;
            }
            QProgressBar::chunk {
                background: qlineargradient(
                    spread:pad, x1:0, y1:0, x2:1, y2:1,
                    stop:0 #8fbc8f, stop:1 #2ecc40
                );
                border-radius: 7px;
            }
        """)
        self.status_progress_layout.addWidget(self.status_label)
        self.status_progress_layout.addStretch()
        self.status_progress_layout.addWidget(self.progress_bar)
        self.right_layout.addLayout(self.status_progress_layout)

        # ====== KHUNG HƯỚNG DẪN ======
        self.guide_widget = QWidget()
        self.guide_layout = QVBoxLayout()
        self.guide_widget.setLayout(self.guide_layout)
        # Tạo toggle cho phần hướng dẫn nhanh
        self.quick_guide_toggle = QToolButton()
        self.quick_guide_toggle.setText("▶ ADB/Fastboot Hướng dẫn nhanh")
        self.quick_guide_toggle.setStyleSheet("QToolButton {font-weight: bold; font-size: 13px; color: #0277bd;}")
        self.quick_guide_toggle.setCheckable(True)
        self.quick_guide_toggle.setChecked(False)
        self.quick_guide_toggle.setToolButtonStyle(Qt.ToolButtonTextOnly)
        self.guide_layout.addWidget(self.quick_guide_toggle)

        self.quick_guide_content = QWidget()
        self.quick_guide_content_layout = QVBoxLayout()
        self.quick_guide_content.setLayout(self.quick_guide_content_layout)
        self.quick_guide_content.setVisible(False)

        code_commands = [
            "adb reboot fastboot",
            "fastboot boot recovery",
            "adb sideload OnePlus5Oxygen_23_OTA_069_all_2010292138_55db5445239246dd.zip",
            "adb sideload OnePlus5TOxygen_43_OTA_069_all_2010292144_76910d123e3940e5.zip",
            "fastboot boot recovery_oneplus5.img",
            "fastboot boot recovery_oneplus5T",
            "fastboot boot twrp-3.4.0-0-cheeseburger_dumpling.img",
            "fastboot boottwrp-3.7.1_12-1-cheeseburger_dumpling.img",
            "fastboot boot twrp-3.7.1_12-2-cheeseburger_dumpling.img"
        ]
        for cmd in code_commands:
            row = QHBoxLayout()
            code_label = QLabel(cmd)
            code_label.setFont(QFont("Consolas", 10))
            code_label.setStyleSheet("""
                QLabel {
                    background: #23272e;
                    color: #f7f7f7;
                    border-radius: 5px;
                    padding: 6px 12px;
                    margin: 3px 0;
                }
                QLabel:hover {
                    background: #343947;
                }
            """)
            code_label.setCursor(QCursor(Qt.PointingHandCursor))
            def make_copy_func(text, label=code_label):
                def copy_code():
                    QApplication.clipboard().setText(text)
                    label.setStyleSheet("""
                        QLabel {
                            background: #3cc34a;
                            color: #fff;
                            border-radius: 5px;
                            padding: 6px 12px;
                            margin: 3px 0;
                        }
                    """)
                    label.setText("Đã copy! ✅")
                    QTimer.singleShot(800, lambda: [
                        label.setText(text),
                        label.setStyleSheet("""
                            QLabel {
                                background: #23272e;
                                color: #f7f7f7;
                                border-radius: 5px;
                                padding: 6px 12px;
                                margin: 3px 0;
                            }
                            QLabel:hover {
                                background: #343947;
                            }
                        """)
                    ])
                return copy_code
            code_label.mousePressEvent = lambda ev, text=cmd, label=code_label: make_copy_func(text, label)()
            row.addWidget(code_label)
            row.addStretch(1)
            self.quick_guide_content_layout.addLayout(row)
        self.guide_layout.addWidget(self.quick_guide_content)

        def toggle_quick_guide():
            if self.quick_guide_toggle.isChecked():
                self.quick_guide_toggle.setText("▼ ADB/Fastboot Hướng dẫn nhanh")
                self.quick_guide_content.setVisible(True)
            else:
                self.quick_guide_toggle.setText("▶ ADB/Fastboot Hướng dẫn nhanh")
                self.quick_guide_content.setVisible(False)
        self.quick_guide_toggle.toggled.connect(toggle_quick_guide)

        self.rom_guide_toggle = QToolButton()
        self.rom_guide_toggle.setText("▶ Hướng dẫn cài ROM chi tiết")
        self.rom_guide_toggle.setStyleSheet("QToolButton {font-weight: bold; font-size: 13px; color: #0277bd;}")
        self.rom_guide_toggle.setCheckable(True)
        self.rom_guide_toggle.setChecked(False)
        self.rom_guide_toggle.setToolButtonStyle(Qt.ToolButtonTextOnly)
        self.guide_layout.addWidget(self.rom_guide_toggle)

        self.rom_guide_content = QWidget()
        self.rom_guide_content_layout = QVBoxLayout()
        self.rom_guide_content.setLayout(self.rom_guide_content_layout)
        self.rom_guide_content.setVisible(False)

        rom_text = """
        <b style='color:red'>***Lưu ý:</b> Bật <b>chế độ nhà phát triển</b> cho các điện thoại.<br>
        <ol>
        <li><b>ROM OxygenOS -->> ROM Tool</b></li>
        <li>Vào chế độ <b>Fastboot</b></li>
        <li>Boot file <b>.img</b></li>
        <li>Copy file sang thiết bị</li>
        <li>Cài <b>zip</b> qua TWRP</li>
        <li>Erase Cache and Data (TWRP)</li>
        <li>Reboot</li>
        </ol>
        <li><b>ROM Tool -->> ROM Tool</b></li>
        <li>1. Vào chế độ <b>Fastboot</b></li>
        <li>2. Boot file <b>.img</b></li>
        <li>3. Wipe Advanced (TWRP) - Xóa Dalvik, Cache, System, Vendor, Data, Internal</li>
        <li>4. Erase Cache and Data (TWRP)</li>
        <li>5. Format Data (TWRP)</li>
        <li>6. Go to Fastboot (TWRP)</li>
        <li>7. Boot file <b>.img</b> lại lần nữa</li>
        <li>8. Copy file sang thiết bị</li>
        <li>9. Cài <b>zip</b> qua TWRP</li>
        <li>10. Erase Cache and Data (TWRP)</li>
        <li>11. Reboot</li>
        """
        self.rom_guide_label = QLabel(rom_text)
        self.rom_guide_label.setWordWrap(True)
        self.rom_guide_label.setStyleSheet("""
            QLabel {
                font-size: 13px;
                background: #f4fbfe;
                border: 1px solid #cce8f6;
                border-radius: 7px;
                padding: 12px;
                margin: 6px 0 0 0;
                color: #263238;
            }
        """)
        self.rom_guide_content_layout.addWidget(self.rom_guide_label)
        self.guide_layout.addWidget(self.rom_guide_content)

        def toggle_rom_guide():
            if self.rom_guide_toggle.isChecked():
                self.rom_guide_toggle.setText("▼ Hướng dẫn cài ROM chi tiết")
                self.rom_guide_content.setVisible(True)
            else:
                self.rom_guide_toggle.setText("▶ Hướng dẫn cài ROM chi tiết")
                self.rom_guide_content.setVisible(False)

        self.rom_guide_toggle.toggled.connect(toggle_rom_guide)

        self.guide_layout.addStretch(1)
        self.guide_widget.setMinimumWidth(350)
        self.guide_widget.setMaximumWidth(650)
        self.guide_widget.hide()

        # ====== QSplitter PHẦN PHẢI: DANH SÁCH THIẾT BỊ <-> HƯỚNG DẪN ======
        self.right_splitter = QSplitter(Qt.Horizontal)
        self.right_splitter.addWidget(self.right_panel)
        self.right_splitter.addWidget(self.guide_widget)

        # ====== MAIN LAYOUT: CHỈ DÙNG QHBoxLayout ======
        main_layout = QHBoxLayout(self)
        main_layout.setContentsMargins(0, 0, 0, 0)
        main_layout.setSpacing(0)
        main_layout.addWidget(self.scroll_area)
        main_layout.addWidget(self.right_splitter, stretch=1)
        self.setLayout(main_layout)

        # ====== KHỞI TẠO BIẾN, TIMER... ======
        self.function_widgets = [
            self.label_I, self.btn_reboot_all, self.btn_adb_reboot_recovery,
            self.label_II, self.btn_reboot_fastboot, self.btn_fastboot_reboot, self.btn_fastboot_reboot_recovery,
            self.label_III, self.label_fastboot_cmd, self.line_fastboot_cmd, self.btn_run_fastboot_cmd,
            self.label_IV, self.label_apk, self.line_apk, self.btn_browse_apk, self.btn_install_apk,
            self.label_V, self.label_img, self.line_img, self.btn_browse_img, self.btn_fastboot_boot,
            self.label_VI, self.label_zip, self.line_zip, self.btn_browse_zip, self.btn_adb_sideload,
            self.label_VII, self.label_push, self.line_push, self.btn_browse_push, self.btn_push_file,
            self.label_VIII, self.label_install_zip, self.line_install_zip, self.btn_install_zip_twrp,
            self.label_IX, self.btn_wipe_advanced, self.btn_wipe_cache_data, self.btn_wipe_data,
            self.label_X, self.btn_twrp_reboot_fastboot, self.btn_twrp_reboot_recovery, self.btn_twrp_reboot,

            # XI: Oxygen to ROM Tool
            self.label_XI,
            self.line_img_xi, self.btn_browse_img_xi,
            self.line_zip5_xi, self.btn_browse_zip5_xi,
            self.line_zip5t_xi, self.btn_browse_zip5t_xi,
            self.btn_start_xi,

            # XII: ROM Tool to Other ROM
            self.label_XII,
            self.line_img_xii, self.btn_browse_img_xii,
            self.line_zip5_xii, self.btn_browse_zip5_xii,
            self.line_zip5t_xii, self.btn_browse_zip5t_xii,
            self.btn_start_xii,
        ]

        self.search_results = []
        self.current_search_index = -1
        self.log_highlight_selections = []
        self.device_item_highlighted = set()
        self.search_navigating = False   # phân biệt “đang gõ” vs “đang nhảy (Next/Enter)”
        self.installEventFilter(self)
        self.apk_file_list = []
        self.push_file_list = []
        self.done = 0
        self.total = 1
        self.progress_lock = threading.Lock()
        self.status_message = ""
        self.last_status = ""
        self.ui_timer = QTimer(self)
        self.ui_timer.timeout.connect(self.update_progress_ui)
        self.ui_timer.start(100)
        self.update_device_list()
        self.auto_scan_timer = QTimer(self)
        self.auto_scan_timer.timeout.connect(self.smart_update_device_list)
        self.auto_scan_timer.start(3000)  # 3 giây
        self.last_device_serials = []


    # ==== BỔ SUNG HÀM GHI LOG  cho QTextEdit ====
    def log_device_result(self, serial, message, status="INFO"):
        self.log_signal.emit(serial, message, status)

    def _append_log(self, serial, message, status):
        import datetime
        # Thêm màu xanh dương nhạt cho log tiến trình TWRP
        color = {
            "OK": "#3cf76b",       # xanh lá
            "ERROR": "#fd6060",    # đỏ
            "INFO": "#ffd600",     # vàng
            "TWRP_LOG": "#80c4ff", # xanh dương nhạt
        }.get(status, "#e2f5fc")   # mặc định xanh rất nhạt

        # Thêm tiền tố >> cho log TWRP (nếu chưa có)
        if status == "TWRP_LOG" and not (message.startswith(">>") or message.startswith("&gt;&gt;")):
            message = f">> {message}"

        now = datetime.datetime.now().strftime("%H:%M:%S")
        html = f'<span style="color:{color}">[{now}] {serial}: {message}</span><br>'
        self.device_log.moveCursor(self.device_log.textCursor().End)
        self.device_log.insertHtml(html)
        self.device_log.ensureCursorVisible()
        self.device_log.moveCursor(self.device_log.textCursor().End)

    # ==== BỔ SUNG HÀM REBOOT FASTBOOT ====
    def fastboot_reboot_selected(self):
        selected_devices = self.get_user_selected_devices()
        def action(serial):
            try:
                cmd = [FASTBOOT, "-s", serial, "reboot"]
                result = run_silent(cmd)
                if result and result.returncode == 0:
                    self.log_device_result(serial, f"Đã reboot (fastboot)", status="OK")
                else:
                    err = result.stderr.strip() if result else "Không rõ"
                    self.log_device_result(serial, f"Lỗi reboot fastboot - {err}", status="ERROR")
            except Exception as e:
                self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
            finally:
                with self.progress_lock:
                    self.done += 1
        self.run_device_action(selected_devices, action, "Đang reboot các thiết bị ở chế độ fastboot...", "Đã reboot (fastboot) cho {n} thiết bị.")

    def fastboot_reboot_recovery_selected(self):
        selected_devices = self.get_user_selected_devices()
        def action(serial):
            try:
                model = get_device_model(serial)
                # Nếu là OnePlus 5/5T thì dùng lệnh oem reboot-recovery để vào recovery gốc
                if model in ["OnePlus 5", "OnePlus 5T"]:
                    cmd = [FASTBOOT, "-s", serial, "oem", "reboot-recovery"]
                    result = run_silent(cmd)
                    if result and result.returncode == 0:
                        self.log_device_result(serial, f"Đã vào recovery gốc (OnePlus)", status="OK")
                    else:
                        err = result.stderr.strip() if result else "Không rõ"
                        self.log_device_result(serial, f"Lỗi vào recovery gốc - {err}", status="ERROR")
                else:
                    cmd = [FASTBOOT, "-s", serial, "reboot", "recovery"]
                    result = run_silent(cmd)
                    if result and result.returncode == 0:
                        self.log_device_result(serial, "Đã reboot recovery (fastboot)", status="OK")
                    else:
                        err = result.stderr.strip() if result else "Không rõ"
                        self.log_device_result(serial, f"Lỗi reboot recovery (fastboot) - {err}", status="ERROR")
            except Exception as e:
                self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
            finally:
                with self.progress_lock:
                    self.done += 1
        self.run_device_action(selected_devices, action, "Đang reboot recovery (fastboot) cho các thiết bị...", "Đã reboot recovery (fastboot) cho {n} thiết bị.")

    def select_user_devices(self):
        """Chọn các thiết bị ở chế độ User (ADB, không phải TWRP)"""
        self.device_list.clearSelection()
        for i in range(self.device_list.count()):
            item = self.device_list.item(i)
            # Chỉ chọn thiết bị có tooltip là "Chế độ ADB" (không có fastboot) và không có [TWRP] trong text
            if item.toolTip() == "Chế độ ADB" and "[TWRP]" not in item.text():
                item.setSelected(True)
        self.update_selected_count()

    def select_sideload_devices(self):
        """Chọn các thiết bị ở chế độ Sideload"""
        self.device_list.clearSelection()
        for i in range(self.device_list.count()):
            item = self.device_list.item(i)
            # Chỉ chọn thiết bị có tooltip là "Chế độ Sideload"
            if item.toolTip() == "Chế độ Sideload":
                item.setSelected(True)
        self.update_selected_count()

    def select_twrp_devices(self):
        """Chọn các thiết bị ở chế độ TWRP (ADB, có [TWRP] trong text)"""
        self.device_list.clearSelection()
        for i in range(self.device_list.count()):
            item = self.device_list.item(i)
            # Chỉ chọn thiết bị có tooltip là "Chế độ ADB" và có [TWRP] trong text
            if item.toolTip() == "Chế độ ADB" and "[TWRP]" in item.text():
                item.setSelected(True)
        self.update_selected_count()

    def adb_push_selected(self):
        push_list = self.push_file_list
        if not push_list:
            self.set_status("Chưa chọn file để copy!")
            self.update_progress(0, 1)
            return
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return

        dialog = CopyProgressDialog(selected_devices, self)
        signal = PushProgressSignal()
        signal.update.connect(dialog.set_percent)
        signal.finished.connect(dialog.accept)
        dialog.setModal(False)
        dialog.show()

        # Giới hạn số thread đồng thời (ví dụ: tối đa 1)
        max_threads = 20
        semaphore = threading.Semaphore(max_threads)

        def work():
            try:
                self.set_status("Đang copy file vào thiết bị được chọn...")
                total = len(selected_devices) * len(push_list)
                with self.progress_lock:
                    self.done = 0
                    self.total = total

                def push(serial):
                    with semaphore:
                        for pushf in push_list:
                            try:
                                cmd = [ADB, "-s", serial, "push", pushf, "/sdcard/"]
                                result = run_silent(cmd)
                                if result and result.returncode == 0:
                                    self.log_device_result(serial, f"Đã copy file: {os.path.basename(pushf)}", status="OK")
                                else:
                                    err = result.stderr.strip() if result else "Không rõ"
                                    self.log_device_result(serial, f"Lỗi copy file: {os.path.basename(pushf)} - {err}", status="ERROR")
                            except Exception as e:
                                self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
                            finally:
                                with self.progress_lock:
                                    self.done += 1
                        # Cập nhật tiến độ cho từng thiết bị
                        signal.update.emit(serial, 100)

                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=push, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã copy {len(push_list)} file cho {len(selected_devices)} thiết bị.")
                with self.progress_lock:
                    self.done = total
                    self.total = total
                signal.finished.emit()
            except Exception as e:
                print("Lỗi tổng thể trong adb_push_selected:", e)
                self.set_status("Có lỗi trong thao tác push!")
                signal.finished.emit()
        self.run_threaded(work)

    def eventFilter(self, obj, event):
        if event.type() == QEvent.KeyPress:
            keyseq = QKeySequence(int(event.modifiers()) + event.key())
            if keyseq.matches(QKeySequence("Ctrl+F")):
                self.show_search_box()
                return True
            elif keyseq.matches(QKeySequence("Esc")):
                if self.search_box_area.isVisible():
                    self.hide_search_box()
                    return True
        return super().eventFilter(obj, event)

    def show_search_box(self):
        self.search_box_area.setVisible(True)
        self.search_box.setFocus()
        self.search_box.selectAll()

    def hide_search_box(self):
        self.search_box_area.setVisible(False)
        self.search_box.clear()
        self.clear_search_highlight()

    def clear_search_highlight(self):
        # Widgets
        for w in self.function_widgets:
            if isinstance(w, (QPushButton, QLabel, QLineEdit)):
                w.setStyleSheet("")

        # Device list items
        for i in range(self.device_list.count()):
            it = self.device_list.item(i)
            it.setBackground(Qt.transparent)
        self.device_item_highlighted.clear()

        # Log
        self.device_log.setExtraSelections([])
        self.log_highlight_selections = []

        # Counter
        self.search_count_label.setText("")
        self.search_results = []
        self.current_search_index = -1

    def highlight_log_occurrences(self, keyword: str):
        """Tô vàng tất cả vị trí match trong QTextEdit (case-insensitive)."""
        self.device_log.setExtraSelections([])
        self.log_highlight_selections = []
        if not keyword:
            return

        text = self.device_log.toPlainText()
        lower = text.lower()
        keyword = keyword.lower()
        start = 0
        from PyQt5.QtGui import QTextCursor, QTextCharFormat, QColor
        from PyQt5.QtWidgets import QTextEdit

        selections = []
        while True:
            idx = lower.find(keyword, start)
            if idx == -1:
                break
            sel = QTextEdit.ExtraSelection()
            cursor = QTextCursor(self.device_log.document())
            cursor.setPosition(idx)
            cursor.setPosition(idx + len(keyword), QTextCursor.KeepAnchor)

            fmt = QTextCharFormat()
            fmt.setBackground(QColor("yellow"))
            sel.cursor = cursor
            sel.format = fmt
            selections.append(sel)
            start = idx + len(keyword)

        self.device_log.setExtraSelections(selections)
        self.log_highlight_selections = selections

    def focus_log_hit(self, hit_index: int):
        """Đưa con trỏ đến kết quả log thứ hit_index trong self.log_highlight_selections."""
        if 0 <= hit_index < len(self.log_highlight_selections):
            cur = self.log_highlight_selections[hit_index].cursor
            self.device_log.setTextCursor(cur)
            self.device_log.ensureCursorVisible()

    def update_search_results(self):
        keyword = self.search_box.text().strip()
        # Reset highlight cũ
        for w in self.function_widgets:
            if isinstance(w, (QPushButton, QLabel, QLineEdit)):
                w.setStyleSheet("")
        for i in range(self.device_list.count()):
            self.device_list.item(i).setBackground(Qt.transparent)
        self.device_item_highlighted.clear()
        self.device_log.setExtraSelections([])
        self.log_highlight_selections = []

        self.search_results = []
        self.current_search_index = -1

        if not keyword:
            self.search_count_label.setText("")
            return

        keylow = keyword.lower()

        # 1) Widgets bên trái
        for w in self.function_widgets:
            text = ""
            if isinstance(w, (QPushButton, QLabel)):
                text = w.text()
            elif isinstance(w, QLineEdit):
                text = w.placeholderText() or w.text()
            if text and keylow in text.lower():
                self.search_results.append({"type": "widget", "ref": w})

        # 2) Bảng danh sách thiết bị
        for i in range(self.device_list.count()):
            it = self.device_list.item(i)
            t = (it.text() or "") + " " + (it.toolTip() or "")
            if keylow in t.lower():
                self.search_results.append({"type": "device", "ref": i})

        # 3) Khung log – tạo highlight tất cả match
        self.highlight_log_occurrences(keylow)
        if self.log_highlight_selections:
            # lưu chỉ số từng hit để có thể nhảy tới
            for idx in range(len(self.log_highlight_selections)):
                self.search_results.append({"type": "log", "ref": idx})

        # Có kết quả → chọn kết quả đầu tiên & tô
        if self.search_results:
            self.current_search_index = 0
            self.highlight_search_results(apply_focus=False)  # chỉ tô, không cướp focus
        self.update_search_counter()


    def goto_next_search(self):
        if not self.search_results:
            return
        self.search_navigating = True
        try:
            self.current_search_index = (self.current_search_index + 1) % len(self.search_results)
            self.highlight_search_results(apply_focus=True)  # cho phép cuộn/nhảy
            self.update_search_counter()
        finally:
            self.search_navigating = False


    def highlight_search_results(self, apply_focus=False):
        # Reset style widgets
        for w in self.function_widgets:
            if isinstance(w, (QPushButton, QLabel, QLineEdit)):
                w.setStyleSheet("")

        # Reset device list item background
        for i in range(self.device_list.count()):
            self.device_list.item(i).setBackground(Qt.transparent)
        self.device_item_highlighted.clear()

        if not self.search_results or self.current_search_index < 0:
            return

        hit = self.search_results[self.current_search_index]
        htype, href = hit["type"], hit["ref"]

        if htype == "widget":
            w = href
            # luôn highlight
            w.setStyleSheet("background: yellow; color: black; font-weight: bold;")
            # chỉ cuộn tới khi điều hướng (Next/Enter); KHÔNG setFocus để khỏi mất caret
            if apply_focus:
                try:
                    self.scroll_area.ensureWidgetVisible(w)
                except Exception:
                    pass

        elif htype == "device":
            idx = href
            it = self.device_list.item(idx)
            it.setBackground(Qt.yellow)
            if apply_focus:
                self.device_list.scrollToItem(it)
                # Không cần đổi focus sang device_list; giữ người dùng ở ô search để bấm Enter tiếp

        elif htype == "log":
            # log_highlight_selections đã có – focus vị trí match khi điều hướng
            if apply_focus:
                self.focus_log_hit(href)
            # Khi chỉ gõ: ExtraSelections đã tô sẵn, không đổi focus khỏi ô search


    def update_search_counter(self):
        total = len(self.search_results)
        cur = (self.current_search_index + 1) if total else 0
        self.search_count_label.setText(f"{cur}/{total}")

    def update_progress_ui(self):
        with self.progress_lock:
            percent = int(self.done * 100 / self.total) if self.total > 0 else 0
            self.progress_bar.setValue(percent)
            if self.status_message != self.last_status:
                self.status_label.setText(self.status_message)
                self.last_status = self.status_message
            total = self.device_list.count()
            selected = len(self.device_list.selectedItems())
            self.device_label.setText(f"Danh sách thiết bị ({selected}/{total}):")

    def set_status(self, message):
        with self.progress_lock:
            self.status_message = message

    def update_progress(self, done, total):
        with self.progress_lock:
            self.done = done
            self.total = max(1, total)

    def select_all_devices(self):
        self.device_list.selectAll()
        self.update_selected_count()

    def deselect_all_devices(self):
        self.device_list.clearSelection()
        self.update_selected_count()

    def update_selected_count(self):
        total = self.device_list.count()
        selected = len(self.device_list.selectedItems())
        self.device_label.setText(f"Danh sách thiết bị ({selected}/{total}):")

    def toggle_guide(self):
        if self.guide_widget.isVisible():
            self.guide_widget.hide()
            self.btn_toggle_guide.setText("Hướng dẫn >>")
        else:
            self.guide_widget.show()
            self.btn_toggle_guide.setText("Hướng dẫn <<")

    def browse_zip(self):
        file, _ = QFileDialog.getOpenFileName(self, "Chọn file ZIP", "", "ZIP Files (*.zip)")
        if file:
            self.line_zip.setText(file)



    def browse_push(self):
        files, _ = QFileDialog.getOpenFileNames(self, "Chọn file để copy", "", "All Files (*)")
        if files:
            self.push_file_list = files
            self.line_push.setText("; ".join(files))

    def browse_apk(self):
        files, _ = QFileDialog.getOpenFileNames(self, "Chọn file APK", "", "APK Files (*.apk)")
        if files:
            self.apk_file_list = files
            self.line_apk.setText("; ".join(files))

    def browse_img(self):
        file, _ = QFileDialog.getOpenFileName(self, "Chọn file IMG", "", "Image Files (*.img)")
        if file:
            self.line_img.setText(file)

        # XI - Oxygen to ROM Tool
    def browse_img_xi(self):
        file, _ = QFileDialog.getOpenFileName(self, "Chọn file IMG cho Oxygen", "", "Image Files (*.img)")
        if file:
            self.line_img_xi.setText(file)

    def browse_zip5_xi(self):
        file, _ = QFileDialog.getOpenFileName(self, "Chọn file ZIP cho OnePlus 5", "", "ZIP Files (*.zip)")
        if file:
            self.line_zip5_xi.setText(file)

    def browse_zip5t_xi(self):
        file, _ = QFileDialog.getOpenFileName(self, "Chọn file ZIP cho OnePlus 5T", "", "ZIP Files (*.zip)")
        if file:
            self.line_zip5t_xi.setText(file)

        # XII - ROM Tool to Other ROM
    def browse_img_xii(self):
        file, _ = QFileDialog.getOpenFileName(self, "Chọn file IMG cho MultiCommand", "", "Image Files (*.img)")
        if file:
            self.line_img_xii.setText(file)

    def browse_zip5_xii(self):
        file, _ = QFileDialog.getOpenFileName(self, "Chọn file ZIP cho OnePlus 5", "", "ZIP Files (*.zip)")
        if file:
            self.line_zip5_xii.setText(file)

    def browse_zip5t_xii(self):
        file, _ = QFileDialog.getOpenFileName(self, "Chọn file ZIP cho OnePlus 5T", "", "ZIP Files (*.zip)")
        if file:
            self.line_zip5t_xii.setText(file)

    def get_user_selected_devices(self):
        res = []
        for item in self.device_list.selectedItems():
            serial = item.text().split(" ", 1)[1].split(" ")[0]
            res.append(serial)
        if not res:
            total = self.device_list.count()
            if total == 0:
                QMessageBox.warning(self, "Thông báo", "Không có thiết bị nào được phát hiện!")
                return []
            reply = QMessageBox.question(
                self,
                "Chưa chọn thiết bị",
                "Bạn chưa chọn thiết bị nào.\nBạn có muốn chạy cho **tất cả các thiết bị** không?",
                QMessageBox.Yes | QMessageBox.Cancel,
                QMessageBox.Cancel
            )
            if reply == QMessageBox.Yes:
                for i in range(total):
                    serial = self.device_list.item(i).text().split(" ", 1)[1].split(" ")[0]
                    res.append(serial)
            else:
                return []
        return res
   

    def run_threaded(self, func):
        threading.Thread(target=func, daemon=True).start()

    def smart_update_device_list(self):
        adb_devices = get_adb_devices()
        fastboot_devices = get_fastboot_devices()
        all_devices = list(sorted(set(adb_devices + fastboot_devices)))
        # So sánh list serial hiện tại với lần trước
        if all_devices != self.last_device_serials:
            # Gọi update_device_list_with_list trước, sau đó mới cập nhật last_device_serials
            self.update_device_list_with_list(all_devices)
            self.last_device_serials = all_devices.copy()
        # Không thay đổi thì không update lại UI, giúp UI mượt mà hơn

    def update_device_list_with_list(self, all_devices):
        old_serials = set(getattr(self, 'last_device_serials', []))
        self.device_list.clear()
        adb_devices = get_adb_devices()
        fastboot_devices = get_fastboot_devices()
        
        # Lấy serial thiết bị ở chế độ sideload
        sideload_serials = []
        try:
            result = subprocess.run(
                [ADB, 'devices'],
                capture_output=True,
                text=True,
                creationflags=getattr(subprocess, "CREATE_NO_WINDOW", 0)
            )
            if result and result.returncode == 0:
                for line in result.stdout.strip().splitlines():
                    if '\t' in line:
                        serial, state = line.split('\t', 1)
                        if state.strip().lower() == 'sideload':
                            sideload_serials.append(serial.strip())
        except Exception:
            pass

        new_serials = set(all_devices) - old_serials
        removed_serials = old_serials - set(all_devices)

        # Log các thiết bị bị ngắt kết nối
        for serial in removed_serials:
            self.log_device_result(serial, f"Thiết bị vừa ngắt kết nối: {serial}", status="ERROR")

        for idx, serial in enumerate(all_devices):
            in_adb = serial in adb_devices
            in_fastboot = serial in fastboot_devices
            is_twrp = False
            if in_adb:
                try:
                    cmd = [ADB, "-s", serial, "shell", "getprop", "ro.twrp.version"]
                    result = run_silent(cmd)
                    if result and result.returncode == 0 and result.stdout.strip():
                        is_twrp = True
                except Exception:
                    pass

            # Đặt tên hiển thị
            if in_fastboot and not in_adb:
                show_model = " (Fastboot)"
            elif in_adb and in_fastboot:
                show_model = " (ADB + Fastboot)"
            elif in_adb:
                model = get_device_model(serial)
                if model == "OnePlus 5":
                    show_model = " (A5000 - OnePlus 5)"
                elif model == "OnePlus 5T":
                    show_model = " (A5010 - OnePlus 5T)"
                elif model and model != "Unknown":
                    show_model = f" ({model})"
                else:
                    show_model = ""
                if is_twrp:
                    show_model += " [TWRP]"
            else:
                show_model = ""

            if serial in sideload_serials:
                show_model += " (Sideload)"

            item_text = f"{idx+1}. {serial}{show_model}"
            item = QListWidgetItem(item_text)
            # Màu cho từng trạng thái
            if in_fastboot and not in_adb:
                item.setForeground(Qt.red)
                item.setToolTip("Chế độ Fastboot")
            elif in_adb and in_fastboot:
                item.setForeground(Qt.blue)
                item.setToolTip("Có cả ADB và Fastboot")
            elif serial in sideload_serials:
                item.setForeground(Qt.darkMagenta)
                item.setToolTip("Chế độ Sideload")
            else:
                item.setForeground(Qt.black)
                item.setToolTip("Chế độ ADB")
            self.device_list.addItem(item)

            if serial in new_serials:
                self.log_device_result(serial, f"Thiết bị vừa kết nối: {serial}", status="INFO")

        self.set_status(f"Đã quét: {len(all_devices)} thiết bị (ADB/Fastboot).")
        self.update_progress(0, 1)
        self.update_selected_count()
        self.update_device_models()   # Cập nhật model sau khi quét xong danh sách!

    def update_device_list(self):
        self.device_list.clear()
        adb_devices = get_adb_devices()
        fastboot_devices = get_fastboot_devices()

        # Lấy serial các thiết bị Sideload từ adb devices
        sideload_serials = []
        try:
            result = subprocess.run(
                [ADB, 'devices'],
                capture_output=True,
                text=True,
                creationflags=getattr(subprocess, "CREATE_NO_WINDOW", 0)
            )
            if result and result.returncode == 0:
                for line in result.stdout.strip().splitlines():
                    if '\t' in line:
                        serial, state = line.split('\t', 1)
                        if state.strip().lower() == 'sideload':
                            sideload_serials.append(serial.strip())
        except Exception:
            pass

        all_devices = list(sorted(set(adb_devices + fastboot_devices)))
        for idx, serial in enumerate(all_devices):
            in_adb = serial in adb_devices
            in_fastboot = serial in fastboot_devices
            is_twrp = False
            if in_adb:
                try:
                    cmd = [ADB, "-s", serial, "shell", "getprop", "ro.twrp.version"]
                    result = run_silent(cmd)
                    if result and result.returncode == 0 and result.stdout.strip():
                        is_twrp = True
                except Exception:
                    pass

            if in_fastboot and not in_adb:
                show_model = " (Fastboot)"
            elif in_adb and in_fastboot:
                show_model = " (ADB + Fastboot)"
            elif in_adb:
                model = get_device_model(serial)
                if model == "OnePlus 5":
                    show_model = " (A5000 - OnePlus 5)"
                elif model == "OnePlus 5T":
                    show_model = " (A5010 - OnePlus 5T)"
                elif model and model != "Unknown":
                    show_model = f" ({model})"
                else:
                    show_model = ""
                if is_twrp:
                    show_model += " [TWRP]"
            else:
                show_model = ""

            if serial in sideload_serials:
                show_model += " (Sideload)"

            item_text = f"{idx+1}. {serial}{show_model}"
            item = QListWidgetItem(item_text)
            if in_fastboot and not in_adb:
                item.setForeground(Qt.red)
                item.setToolTip("Chế độ Fastboot")
            elif in_adb and in_fastboot:
                item.setForeground(Qt.blue)
                item.setToolTip("Có cả ADB và Fastboot")
            elif serial in sideload_serials:
                item.setForeground(Qt.darkMagenta)
                item.setToolTip("Chế độ Sideload")
            else:
                item.setForeground(Qt.black)
                item.setToolTip("Chế độ ADB")
            self.device_list.addItem(item)

        self.set_status(f"Đã quét: {len(all_devices)} thiết bị (ADB/Fastboot).")
        self.update_progress(0, 1)
        self.update_selected_count()
        self.update_device_models()   # GỌI HÀM này sau khi quét xong danh sách!

    def update_device_models(self):
        """
        Cập nhật dict self.device_models với thông tin model mới nhất từ tất cả thiết bị đang kết nối qua ADB.
        """
        self.device_models = {}
        adb_serials = get_adb_devices()
        for serial in adb_serials:
            model = get_device_model(serial)
            self.device_models[serial] = model

    def _resolve_model(self, serial, force_refresh=False):
        """
        Lấy model từ cache; nếu chưa có/Unknown hoặc force_refresh=True
        thì hỏi lại ADB bằng get_device_model() đã gia cố ở trên.
        """
        model = getattr(self, "device_models", {}).get(serial, "")
        if force_refresh or not model or model == "Unknown":
            # dùng hàm robust mới
            model = self.get_device_model(serial) if hasattr(self, "get_device_model") else get_device_model(serial)
            if not hasattr(self, "device_models"):
                self.device_models = {}
            self.device_models[serial] = model or "Unknown"
            self.log_device_result(serial, f"[DEBUG] Cập nhật model: {self.device_models[serial]}", "INFO")
        return model or "Unknown"


    def is_device_oneplus5(self, serial):
        import re
        model = self._resolve_model(serial)
        mu = (model or "").upper()
        # Khớp A5000, "ONEPLUS 5" (có/không khoảng trắng), nhưng KHÔNG ăn vào "5T"
        return bool(
            re.search(r"\bA5000\b", mu)
            or re.search(r"\bONEPLUS\s*5\b", mu)      # "ONEPLUS 5"
            or re.search(r"\bONEPLUS5\b", mu)         # "ONEPLUS5"
        )

    def is_device_oneplus5t(self, serial):
        import re
        model = self._resolve_model(serial)
        mu = (model or "").upper()
        # Khớp A5010, "ONEPLUS 5T" (có/không khoảng trắng)
        return bool(
            re.search(r"\bA5010\b", mu)
            or re.search(r"\bONEPLUS\s*5T\b", mu)     # "ONEPLUS 5T"
            or re.search(r"\bONEPLUS5T\b", mu)        # "ONEPLUS5T"
        )

    def get_zip_path_for_device(self, serial, zip5_path, zip5t_path):
        model = self._resolve_model(serial)  # dùng luôn resolve để cập nhật cache nếu cần
        self.log_device_result(serial, f"[DEBUG] Model={model}, zip5_path={zip5_path}, zip5t_path={zip5t_path}", "INFO")
        # Ưu tiên check 5T trước để tránh mọi khả năng match tràn
        if self.is_device_oneplus5t(serial):
            return zip5t_path
        elif self.is_device_oneplus5(serial):
            return zip5_path
        else:
            self.log_device_result(serial, "Không xác định được model để chọn file zip!", "ERROR")
            return ""

    def select_fastboot_devices(self):
        self.device_list.clearSelection()
        for i in range(self.device_list.count()):
            item = self.device_list.item(i)
            # Chọn nếu tooltip là "Chế độ Fastboot" hoặc "Có cả ADB và Fastboot"
            if "Fastboot" in item.toolTip():
                item.setSelected(True)
        self.update_selected_count()

    def select_adb_devices(self):
        self.device_list.clearSelection()
        for i in range(self.device_list.count()):
            item = self.device_list.item(i)
            # Chỉ chọn thiết bị có tooltip là "Chế độ ADB" (không có fastboot)
            if item.toolTip() == "Chế độ ADB":
                item.setSelected(True)
        self.update_selected_count()

    def adb_reboot_selected(self):
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status("Đang reboot (restart) thiết bị được chọn...")
                n = len(selected_devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = n
                def reboot(serial):
                    try:
                        cmd = [ADB, "-s", serial, "reboot"]
                        result = run_silent(cmd)
                        if result and result.returncode == 0:
                            self.log_device_result(serial, f"Đã reboot (ADB)", status="OK")
                        else:
                            err = result.stderr.strip() if result else "Không rõ"
                            self.log_device_result(serial, f"Lỗi reboot (ADB) - {err}", status="ERROR")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
                    finally:
                        with self.progress_lock:
                            self.done += 1
                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=reboot, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã gửi lệnh reboot cho {n} thiết bị.")
                with self.progress_lock:
                    self.done = n
                    self.total = n
            except Exception as e:
                print("Lỗi tổng thể trong adb_reboot_selected:", e)
                self.set_status("Có lỗi trong thao tác reboot!")
        self.run_threaded(work)

    def adb_reboot_recovery_selected(self):
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status("Đang gửi lệnh ADB reboot recovery cho thiết bị được chọn...")
                n = len(selected_devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = n
                def reboot(serial):
                    try:
                        cmd = [ADB, "-s", serial, "reboot", "recovery"]
                        result = run_silent(cmd)
                        if result and result.returncode == 0:
                            self.log_device_result(serial, f"Đã reboot vào recovery", status="OK")
                        else:
                            err = result.stderr.strip() if result else "Không rõ"
                            self.log_device_result(serial, f"Lỗi reboot recovery - {err}", status="ERROR")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
                    finally:
                        with self.progress_lock:
                            self.done += 1
                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=reboot, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã gửi lệnh reboot recovery cho {n} thiết bị.")
                with self.progress_lock:
                    self.done = n
                    self.total = n
            except Exception as e:
                print("Lỗi tổng thể trong adb_reboot_recovery_selected:", e)
                self.set_status("Có lỗi trong thao tác reboot recovery!")
        self.run_threaded(work)

    def adb_reboot_fastboot(self):
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status("Đang gửi lệnh reboot fastboot thiết bị được chọn...")
                n = len(selected_devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = n
                def reboot(serial):
                    try:
                        cmd = [ADB, "-s", serial, "reboot", "bootloader"]
                        result = run_silent(cmd)
                        if result and result.returncode == 0:
                            self.log_device_result(serial, f"Đã reboot vào fastboot", status="OK")
                        else:
                            err = result.stderr.strip() if result else "Không rõ"
                            self.log_device_result(serial, f"Lỗi reboot fastboot - {err}", status="ERROR")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
                    finally:
                        with self.progress_lock:
                            self.done += 1
                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=reboot, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã gửi lệnh reboot fastboot cho {n} thiết bị.")
                with self.progress_lock:
                    self.done = n
                    self.total = n
            except Exception as e:
                print("Lỗi tổng thể trong adb_reboot_fastboot:", e)
                self.set_status("Có lỗi trong thao tác reboot fastboot!")
        self.run_threaded(work)

    def fastboot_boot_selected(self):
        img = self.line_img.text().strip()
        if not img:
            self.set_status("Chưa chọn file .img!")
            self.update_progress(0, 1)
            return
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status("Đang boot file .img thiết bị được chọn...")
                n = len(selected_devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = n
                def boot(serial):
                    try:
                        cmd = [FASTBOOT, "-s", serial, "boot", img]
                        result = run_silent(cmd)
                        if result and result.returncode == 0:
                            self.log_device_result(serial, f"Đã boot file .img: {os.path.basename(img)}", status="OK")
                        else:
                            err = result.stderr.strip() if result else "Không rõ"
                            self.log_device_result(serial, f"Lỗi boot file .img: {os.path.basename(img)} - {err}", status="ERROR")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
                    finally:
                        with self.progress_lock:
                            self.done += 1
                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=boot, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã boot file .img cho {n} thiết bị.")
                with self.progress_lock:
                    self.done = n
                    self.total = n
            except Exception as e:
                print("Lỗi tổng thể trong fastboot_boot_selected:", e)
                self.set_status("Có lỗi trong thao tác boot fastboot!")
        self.run_threaded(work)

    def run_fastboot_command_selected(self):
        cmdline = self.line_fastboot_cmd.text().strip()
        if not cmdline:
            self.set_status("Chưa nhập lệnh Fastboot!")
            return
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status(f"Đang chạy lệnh: {cmdline}")
                n = len(selected_devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = n
                def runcmd(serial):
                    try:
                        real_cmd = cmdline.replace("$serial", serial).replace("{serial}", serial)
                        if "fastboot" not in real_cmd:
                            real_cmd = f'{FASTBOOT} -s {serial} {real_cmd}'
                        elif "-s" not in real_cmd:
                            parts = real_cmd.split()
                            idx = 1 if parts[0].endswith("fastboot") else 0
                            parts.insert(idx+1, "-s")
                            parts.insert(idx+2, serial)
                            real_cmd = " ".join(parts)
                        result = subprocess.run(real_cmd, shell=True,
                                        timeout=180,
                                        creationflags=getattr(subprocess, "CREATE_NO_WINDOW", 0))
                        if result and result.returncode == 0:
                            self.log_device_result(serial, f"Fastboot command OK: {cmdline}", status="OK")
                        else:
                            err = result.stderr.strip() if result else "Không rõ"
                            self.log_device_result(serial, f"Lỗi fastboot cmd: {cmdline} - {err}", status="ERROR")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
                    finally:
                        with self.progress_lock:
                            self.done += 1
                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=runcmd, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã chạy xong lệnh cho {n} thiết bị!")
                with self.progress_lock:
                    self.done = n
                    self.total = n
            except Exception as e:
                print("Lỗi tổng thể trong run_fastboot_command_selected:", e)
                self.set_status("Có lỗi trong thao tác run fastboot command!")
        self.run_threaded(work)

    def twrp_install_apk(self):
        """
        Flash Magisk hoặc bất kỳ file .apk/.zip trong TWRP.
        - Yêu cầu thiết bị đang ở TWRP.
        - Stream log TWRP theo thời gian thực bằng cách tail -f /tmp/recovery.log.
        - Không reboot, không cài trong system.
        """
        import shlex, threading, subprocess, time

        # ---- Input ----
        name = QInputDialog.getText(
            self, "Flash trong TWRP",
            "Nhập tên file trong /sdcard (vd: Magisk-27.0(27000).apk):"
        )[0].strip()
        if not name:
            self.set_status("Chưa nhập tên file!")
            return
        if not (name.lower().endswith(".apk") or name.lower().endswith(".zip")):
            name += ".apk"

        path = name if name.startswith("/") else f"/sdcard/{name}"

        selected = self.get_user_selected_devices()
        if not selected:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return

        # ---- Helpers ----
        def _run(serial, args):
            return run_silent([ADB, "-s", serial] + args)

        def _q(p):
            return shlex.quote(p)

        def _in_recovery(serial):
            # Nhận diện TWRP đa tiêu chí
            r = _run(serial, ["shell", "getprop", "ro.twrp.boot"])
            if r and r.stdout.strip() == "1":
                return True
            r2 = _run(serial, ["shell", "sh", "-c", "[ -x /sbin/twrp ] && echo OK || echo NO"])
            if r2 and "OK" in (r2.stdout or ""):
                return True
            r3 = _run(serial, ["shell", "sh", "-c", "twrp --version >/dev/null 2>&1 || twrp get time >/dev/null 2>&1; echo $?"])
            if r3 and r3.stdout.strip() == "0":
                return True
            r4 = _run(serial, ["shell", "getprop", "ro.bootmode"])
            mode = (r4.stdout or "").strip().lower() if r4 else ""
            return "recovery" in mode

        def _twrp_mount_data(serial):
            _run(serial, ["shell", "twrp", "mount", "data"])
            chk = _run(serial, ["shell", "mountpoint", "-q", "/data"])
            return bool(chk and chk.returncode == 0)

        def _start_tail_recovery_log(serial, start_marker=None):
            """
            Mở tail -n 0 -f /tmp/recovery.log để stream log realtime.
            Trả về (proc, thread) để caller có thể stop.
            """
            # Đảm bảo file log tồn tại trước khi tail (nếu chưa có, tạo rỗng)
            _run(serial, ["shell", "sh", "-c", "touch /tmp/recovery.log"])

            tail_cmd = [ADB, "-s", serial, "shell", "sh", "-c", "tail -n 0 -f /tmp/recovery.log"]
            proc = subprocess.Popen(
                tail_cmd,
                stdout=subprocess.PIPE,
                stderr=subprocess.STDOUT,
                text=True,
                bufsize=1,
                creationflags=getattr(subprocess, "CREATE_NO_WINDOW", 0)
            )

            def _pump():
                # Optional: đưa marker để dễ cắt log theo phiên
                if start_marker:
                    self.log_device_result(serial, f">> ===== {start_marker} =====", "TWRP_LOG")
                try:
                    for line in iter(proc.stdout.readline, ''):
                        if not line:
                            break
                        line = line.rstrip("\r\n")
                        if line:
                            self.log_device_result(serial, line, "TWRP_LOG")
                except Exception:
                    pass

            t = threading.Thread(target=_pump, daemon=True)
            t.start()
            return proc, t

        def _stop_tail(proc):
            try:
                if proc and proc.poll() is None:
                    proc.terminate()
                    # Nếu chưa chết, kill mạnh tay
                    try:
                        proc.wait(timeout=2)
                    except Exception:
                        proc.kill()
            except Exception:
                pass

        def _twrp_install(serial, rpath):
            """
            Chạy twrp install, trả về (ok, full_output).
            (Không dựa vào buffer của adb; kết quả thực xem ở recovery.log)
            """
            qpath = _q(rpath)
            cmd = f"[ -x /sbin/twrp ] && /sbin/twrp install {qpath} || twrp install {qpath}"
            res = _run(serial, ["shell", "sh", "-c", cmd])
            output = ((res.stdout or "") + "\n" + (res.stderr or "")).strip() if res else ""

            low = output.lower()
            success_markers = [
                "install zip successful",
                "done processing script file",
                "updating partition details...done"
            ]
            error_markers = [
                "failed", "error", "aborted", "invalid zip",
                "zip treble compatibility error", "no such file", "not found", "cannot open"
            ]
            ok = any(m in low for m in success_markers) and not any(e in low for e in error_markers)
            # Lưu ý: thành công thực tế vẫn xem trên recovery.log (đã stream), ở đây chỉ là phụ
            return ok, output

        # ---- Work ----
        def work():
            try:
                self.set_status(f"Đang flash trong TWRP: {path}")
                with self.progress_lock:
                    self.done = 0
                    self.total = len(selected)

                def handle(serial):
                    proc_tail = None
                    try:
                        if not _in_recovery(serial):
                            self.log_device_result(serial, "Thiết bị chưa ở TWRP. Vui lòng boot vào TWRP rồi thử lại.", "ERROR")
                            return
                        if not _twrp_mount_data(serial):
                            self.log_device_result(serial, "Không mount được /data trong TWRP.", "ERROR")
                            return

                        # Bắt đầu tail realtime trước khi install để không bỏ sót dòng đầu
                        proc_tail, _ = _start_tail_recovery_log(serial, start_marker="BEGIN INSTALL")

                        self.log_device_result(serial, f"Đang flash file qua TWRP: {path}", "INFO")
                        ok, out = _twrp_install(serial, path)

                        # Chờ thêm chút để tail hút nốt dòng cuối
                        time.sleep(0.5)
                        _stop_tail(proc_tail); proc_tail = None
                        self.log_device_result(serial, ">> ===== END INSTALL =====", "TWRP_LOG")

                        if ok:
                            self.log_device_result(serial, f"ĐÃ FLASH THÀNH CÔNG: {path}", "OK")
                        else:
                            # Nếu output không đủ tin cậy, user vẫn thấy recovery.log realtime rồi
                            snippet = (out or "Không có output")[-1500:]
                            self.log_device_result(serial, f"LỖI FLASH TWRP: {path}\n---OUTPUT---\n{snippet}", "ERROR")

                    finally:
                        _stop_tail(proc_tail)
                        with self.progress_lock:
                            self.done += 1

                threads = []
                for s in selected:
                    t = threading.Thread(target=handle, args=(s,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()

                self.set_status(f"Hoàn tất flash cho {len(selected)} thiết bị.")
                with self.progress_lock:
                    self.done = self.total

            except Exception as e:
                print("twrp_install_apk error:", e)
                self.set_status("Có lỗi trong thao tác twrp_install_apk!")

        self.run_threaded(work)

    def adb_sideload_selected(self):
        zip_path = self.line_zip.text().strip()
        if not zip_path:
            self.set_status("Chưa chọn file .zip!")
            self.update_progress(0, 1)
            return
        zipf = os.path.basename(zip_path)
        tool_dir = os.path.dirname(os.path.abspath(__file__))
        adb_path = os.path.join(tool_dir, "adb.exe")
        if not os.path.isfile(adb_path):
            self.set_status("Không tìm thấy adb.exe trong thư mục tool! Vui lòng copy adb.exe vào cùng thư mục.")
            return
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            self.set_status("Đang sideload file .zip cho các thiết bị...")
            n = len(selected_devices)
            with self.progress_lock:
                self.done = 0
                self.total = n
            def sideload(serial):
                try:
                    cmd = [adb_path, "-s", serial, "sideload", zip_path]
                    result = run_silent(cmd, timeout=100000)
                    if result and result.returncode == 0:
                        self.log_device_result(serial, f"Sideload thành công: {zipf}", status="OK")
                    else:
                        err = result.stderr.strip() if result else "Không rõ"
                        self.log_device_result(serial, f"Lỗi sideload: {zipf} - {err}", status="ERROR")
                except Exception as e:
                    self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
                finally:
                    with self.progress_lock:
                        self.done += 1
            threads = []
            for serial in selected_devices:
                t = threading.Thread(target=sideload, args=(serial,))
                threads.append(t)
                t.start()
            for t in threads:
                t.join()
            self.set_status(f"Đã sideload file .zip cho {n} thiết bị.")
            with self.progress_lock:
                self.done = n
                self.total = n
        self.run_threaded(work)

    def adb_install_apk_selected(self):
        apk_list = self.apk_file_list
        if not apk_list:
            self.set_status("Chưa chọn file APK!")
            self.update_progress(0, 1)
            return
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status("Đang cài APK vào thiết bị được chọn...")
                total = len(selected_devices) * len(apk_list)
                with self.progress_lock:
                    self.done = 0
                    self.total = total
                def install(serial):
                    for apk in apk_list:
                        try:
                            cmd = [ADB, "-s", serial, "install", "-r", apk]
                            result = run_silent(cmd)
                            if result and result.returncode == 0:
                                self.log_device_result(serial, f"Đã cài APK: {os.path.basename(apk)}", status="OK")
                            else:
                                err = result.stderr.strip() if result else "Không rõ"
                                self.log_device_result(serial, f"Lỗi cài APK: {os.path.basename(apk)} - {err}", status="ERROR")
                        except Exception as e:
                            self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
                        finally:
                            with self.progress_lock:
                                self.done += 1
                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=install, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã cài {len(apk_list)} APK cho {len(selected_devices)} thiết bị.")
                with self.progress_lock:
                    self.done = total
                    self.total = total
            except Exception as e:
                print("Lỗi tổng thể trong adb_install_apk_selected:", e)
                self.set_status("Có lỗi trong thao tác install APK!")
        self.run_threaded(work)

    def twrp_install_zip(self):
        zipname = self.line_install_zip.text().strip()
        if not zipname:
            self.set_status("Chưa nhập tên file zip!")
            return
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status(f"Đang cài đặt file ZIP: {zipname} qua TWRP...")
                n = len(selected_devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = n
                def install(serial):
                    try:
                        if not zipname.endswith(".zip"):
                            filename = zipname + ".zip"
                        else:
                            filename = zipname
                        path = "/sdcard/" + filename if not filename.startswith("/") else filename
                        cmd = [ADB, "-s", serial, "shell", "twrp", "install", path]
                        result = run_silent(cmd)
                        if result and result.returncode == 0:
                            self.log_device_result(serial, f"Đã cài zip qua TWRP: {os.path.basename(path)}", status="OK")
                        else:
                            err = result.stderr.strip() if result else "Không rõ"
                            self.log_device_result(serial, f"Lỗi cài zip TWRP: {os.path.basename(path)} - {err}", status="ERROR")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
                    finally:
                        with self.progress_lock:
                            self.done += 1
                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=install, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã gửi lệnh cài ZIP cho {n} thiết bị (vào TWRP).")
                with self.progress_lock:
                    self.done = n
                    self.total = n
            except Exception as e:
                print("Lỗi tổng thể trong twrp_install_zip:", e)
                self.set_status("Có lỗi trong thao tác install zip TWRP!")
        self.run_threaded(work)

    def twrp_wipe_cache_data(self):
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status("Đang xóa cache & data (TWRP)...")
                n = len(selected_devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = n * 2
                def wipe(serial):
                    for p in ["cache", "data"]:
                        try:
                            cmd = [ADB, "-s", serial, "shell", "twrp", "wipe", p]
                            result = run_silent(cmd)
                            if result and result.returncode == 0:
                                self.log_device_result(serial, f"Wipe {p} thành công (TWRP)", status="OK")
                            else:
                                err = result.stderr.strip() if result else "Không rõ"
                                self.log_device_result(serial, f"Lỗi wipe {p} (TWRP) - {err}", status="ERROR")
                        except Exception as e:
                            self.log_device_result(serial, f"Lỗi exception wipe {p}: {e}", status="ERROR")
                    # Nếu là OnePlus 5/5T thì xóa sạch dữ liệu thủ công
                    model = get_device_model(serial)
                    if model in ["OnePlus 5", "OnePlus 5T"]:
                        try:
                            cmds = [
                                [ADB, "-s", serial, "shell", "umount /data || true"],
                                [ADB, "-s", serial, "shell", "umount /cache || true"],
                                [ADB, "-s", serial, "shell", "rm -rf /data/* /data/.??*"],
                                [ADB, "-s", serial, "shell", "rm -rf /cache/* /cache/.??*"]
                            ]
                            for cmdx in cmds:
                                run_silent(cmdx)
                            self.log_device_result(serial, "Đã xóa sạch dữ liệu (OnePlus 5/5T)", status="OK")
                        except Exception as e:
                            self.log_device_result(serial, f"Lỗi xóa sạch dữ liệu: {e}", status="ERROR")
                    with self.progress_lock:
                        self.done += 2
                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=wipe, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã xóa cache & data cho {n} thiết bị.")
                with self.progress_lock:
                    self.done = n * 2
                    self.total = n * 2
            except Exception as e:
                print("Lỗi tổng thể trong twrp_wipe_cache_data:", e)
                self.set_status("Có lỗi trong thao tác wipe cache/data TWRP!")
        self.run_threaded(work)

    # def twrp_wipe_advanced(self):
    #     selected_devices = self.get_user_selected_devices()
    #     if not selected_devices:
    #         self.set_status("Không có thiết bị nào được chọn.")
    #         self.update_progress(0, 1)
    #         return
    #     def work():
    #         try:
    #             self.set_status("Đang thực hiện Wipe Advanced (TWRP)...")
    #             n = len(selected_devices)
    #             partitions = ["dalvik", "cache", "system", "vendor", "data", "internal"]
    #             with self.progress_lock:
    #                 self.done = 0
    #                 self.total = n * len(partitions)
    #             def wipe(serial):
    #                 for p in partitions:
    #                     try:
    #                         cmd = [ADB, "-s", serial, "shell", "twrp", "wipe", p]
    #                         result = run_silent(cmd)
    #                         if result and result.returncode == 0:
    #                             self.log_device_result(serial, f"Wipe {p} thành công (TWRP)", status="OK")
    #                         else:
    #                             err = result.stderr.strip() if result else "Không rõ"
    #                             self.log_device_result(serial, f"Lỗi wipe {p} (TWRP) - {err}", status="ERROR")
    #                     except Exception as e:
    #                         self.log_device_result(serial, f"Lỗi exception wipe {p}: {e}", status="ERROR")
    #                 with self.progress_lock:
    #                     self.done += len(partitions)
    #             threads = []
    #             for serial in selected_devices:
    #                 t = threading.Thread(target=wipe, args=(serial,))
    #                 threads.append(t)
    #                 t.start()
    #             for t in threads:
    #                 t.join()
    #             self.set_status(f"Đã thực hiện Wipe Advanced cho {n} thiết bị.")
    #             with self.progress_lock:
    #                 self.done = n * len(partitions)
    #                 self.total = n * len(partitions)
    #         except Exception as e:
    #             print("Lỗi tổng thể trong twrp_wipe_advanced:", e)
    #             self.set_status("Có lỗi trong thao tác wipe advanced TWRP!")
    #     self.run_threaded(work)

    def twrp_reboot_fastboot(self):
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status("Đang gửi lệnh vào Fastboot (TWRP)...")
                n = len(selected_devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = n
                def reboot_fastboot(serial):
                    try:
                        cmd = [ADB, "-s", serial, "shell", "twrp", "reboot", "bootloader"]
                        result = run_silent(cmd)
                        if result and result.returncode == 0:
                            self.log_device_result(serial, f"TWRP reboot vào fastboot OK", status="OK")
                        else:
                            err = result.stderr.strip() if result else "Không rõ"
                            self.log_device_result(serial, f"Lỗi TWRP reboot fastboot - {err}", status="ERROR")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
                    finally:
                        with self.progress_lock:
                            self.done += 1
                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=reboot_fastboot, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã gửi lệnh vào Fastboot (TWRP) cho {n} thiết bị.")
                with self.progress_lock:
                    self.done = n
                    self.total = n
            except Exception as e:
                print("Lỗi tổng thể trong twrp_reboot_fastboot:", e)
                self.set_status("Có lỗi trong thao tác twrp reboot fastboot!")
        self.run_threaded(work)

    def twrp_reboot_recovery(self):
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status("Đang gửi lệnh vào Recovery (TWRP)...")
                n = len(selected_devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = n
                def reboot_recovery(serial):
                    try:
                        cmd = [ADB, "-s", serial, "shell", "twrp", "reboot", "recovery"]
                        result = run_silent(cmd)
                        if result and result.returncode == 0:
                            self.log_device_result(serial, f"TWRP reboot vào recovery OK", status="OK")
                        else:
                            err = result.stderr.strip() if result else "Không rõ"
                            self.log_device_result(serial, f"Lỗi TWRP reboot recovery - {err}", status="ERROR")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
                    finally:
                        with self.progress_lock:
                            self.done += 1
                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=reboot_recovery, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã gửi lệnh vào Recovery (TWRP) cho {n} thiết bị.")
                with self.progress_lock:
                    self.done = n
                    self.total = n
            except Exception as e:
                print("Lỗi tổng thể trong twrp_reboot_recovery:", e)
                self.set_status("Có lỗi trong thao tác twrp reboot recovery!")
        self.run_threaded(work)

    def twrp_reboot(self):
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status("Đang reboot lại thiết bị từ TWRP...")
                n = len(selected_devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = n
                def reboot(serial):
                    try:
                        cmd = [ADB, "-s", serial, "shell", "twrp", "reboot"]
                        result = run_silent(cmd)
                        if result and result.returncode == 0:
                            self.log_device_result(serial, f"Reboot từ TWRP OK", status="OK")
                        else:
                            err = result.stderr.strip() if result else "Không rõ"
                            self.log_device_result(serial, f"Lỗi reboot từ TWRP - {err}", status="ERROR")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi exception: {e}", status="ERROR")
                    finally:
                        with self.progress_lock:
                            self.done += 1
                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=reboot, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã reboot lại {n} thiết bị từ TWRP.")
                with self.progress_lock:
                    self.done = n
                    self.total = n
            except Exception as e:
                print("Lỗi tổng thể trong twrp_reboot:", e)
                self.set_status("Có lỗi trong thao tác twrp reboot!")
        self.run_threaded(work)

    def twrp_wipe_data(self):
        selected_devices = self.get_user_selected_devices()
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            self.update_progress(0, 1)
            return
        def work():
            try:
                self.set_status("Đang format Data (TWRP)...")
                n = len(selected_devices)
                with self.progress_lock:
                    self.done = 0
                    self.total = n
                    def wipe(serial):
                        try:
                            cmd = [ADB, "-s", serial, "shell", "mke2fs", "-t", "ext4", "/dev/block/bootdevice/by-name/userdata"]
                            result = run_silent(cmd)
                            if result and result.returncode == 0:
                                self.log_device_result(serial, f"Format Data thành công (TWRP)", status="OK")
                            else:
                                err = result.stderr.strip() if result else "Không rõ"
                                self.log_device_result(serial, f"Lỗi format Data (TWRP) - {err}", status="ERROR")

                            # Thêm xóa cache cho tất cả thiết bị
                            cache_cmds = [
                                [ADB, "-s", serial, "shell", "twrp wipe cache"],
                                [ADB, "-s", serial, "shell", "rm -rf /cache/*"],
                                [ADB, "-s", serial, "shell", "rm -rf /cache/.??*"]
                            ]
                            for cmdx in cache_cmds:
                                run_silent(cmdx)

                            # Nếu là OnePlus 5/5T thì xóa sạch dữ liệu thủ công
                            model = get_device_model(serial)
                            if model in ["OnePlus 5", "OnePlus 5T"]:
                                try:
                                    cmds = [
                                        [ADB, "-s", serial, "shell", "umount /data || true"],
                                        [ADB, "-s", serial, "shell", "umount /cache || true"],
                                        [ADB, "-s", serial, "shell", "rm -rf /data/* /data/.??*"],
                                        [ADB, "-s", serial, "shell", "rm -rf /cache/* /cache/.??*"],
                                        [ADB, "-s", serial, "shell", "rm -rf /sdcard/* /sdcard/.??*"]
                                    ]
                                    for cmdx in cmds:
                                        run_silent(cmdx)
                                    self.log_device_result(serial, "Đã xóa sạch dữ liệu (OnePlus 5/5T)", status="OK")
                                except Exception as e:
                                    self.log_device_result(serial, f"Lỗi xóa sạch dữ liệu: {e}", status="ERROR")
                        except Exception as e:
                            self.log_device_result(serial, f"Lỗi exception format Data: {e}", status="ERROR")
                        finally:
                            with self.progress_lock:
                                self.done += 1
                threads = []
                for serial in selected_devices:
                    t = threading.Thread(target=wipe, args=(serial,))
                    threads.append(t)
                    t.start()
                for t in threads:
                    t.join()
                self.set_status(f"Đã format Data cho {n} thiết bị.")
                with self.progress_lock:
                    self.done = n
                    self.total = n
            except Exception as e:
                print("Lỗi tổng thể trong twrp_wipe_data:", e)
                self.set_status("Có lỗi trong thao tác format data TWRP!")
        self.run_threaded(work)

    def multi_command_rom_oxygen_to_rom_tool(self):
        selected_devices = self.get_user_selected_devices()
        img_path = self.line_img_xi.text().strip()
        zip5_path = self.line_zip5_xi.text().strip()
        zip5t_path = self.line_zip5t_xi.text().strip()

        # Validate input
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            return
        if not img_path or not os.path.isfile(img_path):
            self.set_status("Chưa chọn file .img hợp lệ!")
            return
        if not zip5_path or not os.path.isfile(zip5_path):
            self.set_status("Chưa chọn file .zip cho 5 hợp lệ!")
            return
        if not zip5t_path or not os.path.isfile(zip5t_path):
            self.set_status("Chưa chọn file .zip cho 5T hợp lệ!")
            return

        self.set_status("Đang thực hiện MultiCommand ROM Oxygen to ROM Tool...")

        def run_for_device(serial):
            steps_ok = True
            try:
                # B1: Boot file .img vào TWRP
                self.log_device_result(serial, "Bước 1: Boot file .img vào TWRP", "INFO")
                try:
                    self.fastboot_boot_single(serial, img_path)
                except Exception as e:
                    self.log_device_result(serial, f"Lỗi boot .img: {e}", "ERROR")
                    steps_ok = False

                # B2: Chờ ADB recovery (TWRP)
                if steps_ok:
                    self.log_device_result(serial, "Chờ thiết bị vào TWRP (ADB)...", "INFO")
                    found = False
                    for _ in range(90):   # tối đa ~45s
                        if serial in get_adb_devices():
                            found = True
                            break
                        time.sleep(0.5)
                    if found:
                        time.sleep(5)
                    else:
                        self.log_device_result(serial, "Không thấy thiết bị (ADB) sau khi boot .img", "ERROR")
                        steps_ok = False

                # B3: LÚC NÀY MỚI XÁC ĐỊNH MODEL & CHỌN ZIP
                if steps_ok:
                    model = self._resolve_model(serial, force_refresh=True)
                    # debug thêm
                    self.log_device_result(serial, f"[DEBUG] Model sau khi vào TWRP: {model}", "INFO")
                    if "5T" in model or self.is_device_oneplus5t(serial):
                        zip_path = zip5t_path
                    elif "5" in model or self.is_device_oneplus5(serial):
                        zip_path = zip5_path
                    else:
                        # thử lần cuối với codenames
                        if "dumpling" in model.lower():
                            zip_path = zip5t_path
                        elif "cheeseburger" in model.lower():
                            zip_path = zip5_path
                        else:
                            self.log_device_result(serial, "Không xác định được model để chọn file zip!", "ERROR")
                            steps_ok = False
                            zip_path = ""

                # B4: Copy file .zip vào máy
                if steps_ok and zip_path:
                    self.log_device_result(serial, f"Bước 2: Copy file .zip vào thiết bị ({zip_path})", "INFO")
                    try:
                        self.adb_push_single(serial, zip_path)
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi copy .zip: {e}", "ERROR")
                        steps_ok = False

                # B5: Cài đặt file .zip qua TWRP
                if steps_ok and zip_path:
                    self.log_device_result(serial, "Bước 3: Cài file .zip qua TWRP", "INFO")
                    try:
                        self.twrp_install_zip_single(serial, zip_path)
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi cài file .zip: {e}", "ERROR")
                        steps_ok = False

                # B6: Wipe cache/data
                if steps_ok:
                    try:
                        self.log_device_result(serial, "Bước 4: Wipe Cache/Data", "INFO")
                        self.twrp_wipe_cache_data_single(serial)
                        self.twrp_wipe_data_single(serial)
                        self.log_device_result(serial, "Đã wipe Cache/Data", "OK")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi wipe cache/data: {e}", "ERROR")
                        steps_ok = False

                # B7: Reboot
                if steps_ok:
                    try:
                        self.log_device_result(serial, "Bước 5: Reboot lại thiết bị", "INFO")
                        self.twrp_reboot_single(serial)
                        self.log_device_result(serial, "Hoàn thành quy trình!", "OK")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi reboot: {e}", "ERROR")

            except Exception as e:
                self.log_device_result(serial, f"Lỗi không xác định: {e}", "ERROR")

        for serial in selected_devices:
            threading.Thread(target=run_for_device, args=(serial,), daemon=True).start()



    def multi_command_rom_tool_to_other_rom(self):
        img_path = self.line_img_xii.text().strip()
        zip5_path = self.line_zip5_xii.text().strip()
        zip5t_path = self.line_zip5t_xii.text().strip()
        selected_devices = self.get_user_selected_devices()
        if not img_path or not zip5_path or not zip5t_path:
            self.set_status("Vui lòng chọn đủ file .img, .zip cho 5 và .zip cho 5T!")
            return
        if not selected_devices:
            self.set_status("Không có thiết bị nào được chọn.")
            return

        # Cập nhật lại models nếu cần (đảm bảo self.device_models[serial] đã đúng)
        self.update_device_models()

        def run_for_device(serial):
            steps_ok = True

            # Chọn file .zip đúng với thiết bị
            zip_path = zip5_path if self.is_device_oneplus5(serial) else zip5t_path

            try:
                # 1. Boot file .img
                if steps_ok:
                    self.log_device_result(serial, "Bước 1: Boot file .img", "INFO")
                    try:
                        self.fastboot_boot_single(serial, img_path)
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi boot .img: {e}", "ERROR")
                        steps_ok = False

                # Chờ thiết bị kết nối lại qua ADB sau boot .img lần 1
                if steps_ok:
                    max_wait = 30
                    found = False
                    for waited in range(max_wait * 2):
                        if serial in get_adb_devices():
                            found = True
                            break
                        time.sleep(0.5)
                    if found:
                        time.sleep(10)
                    else:
                        self.log_device_result(serial, "Không phát hiện thiết bị kết nối lại sau boot .img lần 1!", "ERROR")
                        steps_ok = False

                # 2. Wipe advanced (TWRP)
                if steps_ok:
                    self.log_device_result(serial, "Bước 2: Wipe advanced (TWRP)", "INFO")
                    try:
                        self.twrp_wipe_advanced_single(serial)
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi wipe advanced: {e}", "ERROR")
                        steps_ok = False
                
                # # 3. Wipe cache/data
                # if steps_ok:
                #     self.log_device_result(serial, "Bước 3: Wipe cache (TWRP)", "INFO")
                #     try:
                #         self.twrp_wipe_cache_data_single(serial)
                #         self.log_device_result(serial, "Bước 4: Format data (TWRP)", "INFO")
                #         self.twrp_wipe_data_single(serial)
                #     except Exception as e:
                #         self.log_device_result(serial, f"Lỗi wipe/cache data: {e}", "ERROR")
                #         steps_ok = False

                # 4. Go to Fastboot (TWRP)
                if steps_ok:
                    self.log_device_result(serial, "Bước 5: Reboot to Fastboot (TWRP)", "INFO")
                    try:
                        self.twrp_reboot_fastboot_single(serial)
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi reboot fastboot: {e}", "ERROR")
                        steps_ok = False

                # Chờ thiết bị kết nối lại qua Fastboot
                if steps_ok:
                    max_wait = 30
                    found = False
                    for waited in range(max_wait * 2):
                        if serial in get_fastboot_devices():
                            found = True
                            break
                        time.sleep(0.5)
                    if found:
                        time.sleep(10)
                    else:
                        self.log_device_result(serial, "Không phát hiện thiết bị vào Fastboot sau reboot từ TWRP!", "ERROR")
                        steps_ok = False

                # 5. Boot file .img lần 2
                if steps_ok:
                    self.log_device_result(serial, "Bước 6: Boot file .img lần 2", "INFO")
                    try:
                        self.fastboot_boot_single(serial, img_path)
                        self.log_device_result(serial, "Đã boot file .img lần 2 xong", "OK")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi boot .img lần 2: {e}", "ERROR")
                        steps_ok = False

                # Chờ thiết bị kết nối lại ADB sau boot lần 2
                if steps_ok:
                    max_wait = 30
                    found = False
                    for waited in range(max_wait * 2):
                        if serial in get_adb_devices():
                            found = True
                            break
                        time.sleep(0.5)
                    if found:
                        time.sleep(10)
                    else:
                        self.log_device_result(serial, "Không phát hiện thiết bị kết nối lại sau boot .img lần 2!", "ERROR")
                        steps_ok = False

                # 6. Copy file .zip vào thiết bị
                if steps_ok:
                    if self.is_device_oneplus5t(serial):
                        # Đối với 5T, giữ nguyên zip_path đã lấy từ đầu
                        self.log_device_result(serial, f"Bước 7: Copy file .zip vào thiết bị 5T ({zip_path})", "INFO")
                    else:
                        # Đối với các máy khác, lấy lại zip_path rồi copy
                        zip_path = self.get_zip_path_for_device(serial, zip5_path, zip5t_path)
                        self.log_device_result(serial, f"Bước 7: Copy file .zip vào thiết bị ({zip_path})", "INFO")
                    try:
                        self.adb_push_single(serial, zip_path)
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi copy .zip: {e}", "ERROR")
                        steps_ok = False

                # 7. Cài file .zip qua TWRP
                if steps_ok:
                    if self.is_device_oneplus5t(serial):
                        # Đối với 5T, giữ nguyên zip_path đã lấy từ đầu
                        self.log_device_result(serial, f"Bước 8: Cài file .zip qua TWRP ({zip_path})", "INFO")
                    else:
                        # Lấy lại zip_path một lần nữa để đảm bảo đúng file (có thể bỏ nếu chắc chắn không thay đổi)
                        zip_path = self.get_zip_path_for_device(serial, zip5_path, zip5t_path)
                        self.log_device_result(serial, f"Bước 8: Cài file .zip qua TWRP ({zip_path})", "INFO")
                    try:
                        self.twrp_install_zip_single(serial, zip_path)
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi cài zip: {e}", "ERROR")
                        steps_ok = False

                # 8. Wipe cache/data lần 2
                if steps_ok:
                    self.log_device_result(serial, "Bước 9: Wipe cache (TWRP) lần 2", "INFO")
                    try:
                        self.twrp_wipe_cache_data_single(serial)
                        self.log_device_result(serial, "Bước 10: Format data (TWRP) lần 2", "INFO")
                        self.twrp_wipe_data_single(serial)
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi wipe/cache data lần 2: {e}", "ERROR")
                        steps_ok = False

                # 9. Reboot lại
                if steps_ok:
                    self.log_device_result(serial, "Bước 11: Reboot lại thiết bị", "INFO")
                    try:
                        self.twrp_reboot_single(serial)
                        self.log_device_result(serial, "Hoàn thành quy trình!", "OK")
                    except Exception as e:
                        self.log_device_result(serial, f"Lỗi reboot: {e}", "ERROR")
            except Exception as e:
                self.log_device_result(serial, f"Lỗi không xác định: {e}", "ERROR")

        # Chạy song song cho từng thiết bị
        for serial in selected_devices:
            threading.Thread(target=run_for_device, args=(serial,), daemon=True).start()

    def fastboot_boot_single(self, serial, img_path):
        try:
            cmd = [FASTBOOT, "-s", serial, "boot", img_path]
            result = run_silent(cmd)
            if result and result.returncode == 0:
                self.log_device_result(serial, f"Boot .img thành công", "OK")
            else:
                err = result.stderr.strip() if result else "Không rõ"
                self.log_device_result(serial, f"Lỗi boot .img - {err}", "ERROR")
        except Exception as e:
            self.log_device_result(serial, f"Lỗi boot .img: {e}", "ERROR")

    def twrp_wipe_advanced_single(self, serial):
        steps = [
            ("Wipe Dalvik",       [ADB, "-s", serial, "shell", "twrp wipe dalvik"]),
            # ("Wipe Cache",        [ADB, "-s", serial, "shell", "twrp wipe cache"]),
            ("Wipe Data",         [ADB, "-s", serial, "shell", "twrp wipe data"]),
            ("Format Data",       [ADB, "-s", serial, "shell", "twrp format data"]),
            ("Wipe System",       [ADB, "-s", serial, "shell", "twrp wipe system"]),        # TWRP 3.x có hỗ trợ lệnh này
            ("Format System",     [ADB, "-s", serial, "shell", "twrp format system"]),      # TWRP một số bản hỗ trợ
            # ("Xóa system thủ công", [ADB, "-s", serial, "shell", "rm -rf /system/*"]),
            # ("Xóa vendor thủ công", [ADB, "-s", serial, "shell", "rm -rf /vendor/*"]),
        ]
        for step_name, cmd in steps:
            result, error = run_with_timeout(cmd, timeout=20)
            if error:
                self.log_device_result(serial, f"{step_name} - {error}", "ERROR")
            elif result and result.returncode == 0:
                self.log_device_result(serial, f"{step_name} thành công", "OK")
            else:
                err = result.stderr.strip() if result else "Không rõ"
                self.log_device_result(serial, f"{step_name} lỗi - {err}", "ERROR")
        self.log_device_result(serial, "Wipe advanced (TWRP) hoàn tất", "OK")

    def twrp_wipe_data_single(self, serial):
        try:
            self.log_device_result(serial, "Bắt đầu format data...", "INFO")
            # Format data qua TWRP (xóa luôn cả Internal Storage nếu TWRP hỗ trợ)
            cmd_format = [ADB, "-s", serial, "shell", "twrp format data"]
            run_silent(cmd_format)

            # Xóa thủ công thư mục bộ nhớ trong để chắc chắn sạch hoàn toàn
            cmds = [
                [ADB, "-s", serial, "shell", "rm -rf /data/media/*"],
                [ADB, "-s", serial, "shell", "rm -rf /sdcard/*"],
            ]
            for c in cmds:
                run_silent(c)

            # Xóa sạch cache
            cache_cmds = [
                [ADB, "-s", serial, "shell", "twrp wipe cache"],
                [ADB, "-s", serial, "shell", "rm -rf /cache/*"],
                [ADB, "-s", serial, "shell", "rm -rf /cache/.??*"]
            ]
            for c in cache_cmds:
                run_silent(c)

            # Deep wipe cho OnePlus 5/5T nếu cần
            model = get_device_model(serial)
            if model in ["OnePlus 5", "OnePlus 5T"]:
                try:
                    cmds_op = [
                        [ADB, "-s", serial, "shell", "umount /data"],
                        [ADB, "-s", serial, "shell", "rm -rf /data/*"],
                    ]
                    for c in cmds_op:
                        run_silent(c)
                    self.log_device_result(serial, "Đã format & wipe sạch dữ liệu (OnePlus 5/5T)", "OK")
                except Exception as e:
                    self.log_device_result(serial, f"Lỗi format & wipe sạch dữ liệu OnePlus 5/5T: {e}", "ERROR")

            self.log_device_result(serial, "Format data (TWRP), xóa bộ nhớ trong và cache xong", "OK")
        except Exception as e:
            self.log_device_result(serial, f"Lỗi format data: {e}", "ERROR")

    def twrp_wipe_cache_data_single(self, serial):
        try:
            cmd = [ADB, "-s", serial, "shell", "twrp wipe cache"]
            run_silent(cmd)
            cmd = [ADB, "-s", serial, "shell", "twrp wipe data"]
            run_silent(cmd)
            # Deep wipe for OnePlus 5/5T
            model = get_device_model(serial)
            if model in ["OnePlus 5", "OnePlus 5T"]:
                try:
                    # Unmount and remove cache/data manually
                    cmds = [
                        [ADB, "-s", serial, "shell", "umount /data"],
                        [ADB, "-s", serial, "shell", "umount /cache"],
                        [ADB, "-s", serial, "shell", "rm -rf /data/*"],
                        [ADB, "-s", serial, "shell", "rm -rf /cache/*"]
                    ]
                    for c in cmds:
                        run_silent(c)
                    self.log_device_result(serial, "Đã wipe sạch dữ liệu (OnePlus 5/5T)", "OK")
                except Exception as e:
                    self.log_device_result(serial, f"Lỗi wipe sạch dữ liệu OnePlus 5/5T: {e}", "ERROR")
            self.log_device_result(serial, "Wipe cache/data (TWRP) xong", "OK")
        except Exception as e:
            self.log_device_result(serial, f"Lỗi wipe cache/data: {e}", "ERROR")

    def twrp_reboot_fastboot_single(self, serial):
        try:
            cmd = [ADB, "-s", serial, "shell", "twrp reboot bootloader"]
            run_silent(cmd)
            self.log_device_result(serial, "Reboot to Fastboot (TWRP) xong", "OK")
        except Exception as e:
            self.log_device_result(serial, f"Lỗi reboot fastboot: {e}", "ERROR")

    def adb_push_single(self, serial, zip_path):
        try:
            self.set_status(f"Đang copy file {os.path.basename(zip_path)} vào thiết bị {serial}...")
            cmd = [ADB, "-s", serial, "push", zip_path, "/sdcard/"]
            result = run_silent(cmd)
            if result and result.returncode == 0:
                self.log_device_result(serial, "Copy file .zip xong", "OK")
                time.sleep(2)
            else:
                err = result.stderr.strip() if result else "Không rõ"
                self.log_device_result(serial, f"Lỗi copy .zip: {err}", "ERROR")
        except Exception as e:
            self.log_device_result(serial, f"Lỗi copy .zip: {e}", "ERROR")

    def twrp_install_zip_single(self, serial, zip_path):
        try:
            zip_name = os.path.basename(zip_path)
            cmd = [ADB, "-s", serial, "shell", f"twrp install /sdcard/{zip_name}"]
            result = run_silent(cmd)
            if result:
                self.log_device_result(serial, f"stdout: {result.stdout.strip()}", "DEBUG")
                self.log_device_result(serial, f"stderr: {result.stderr.strip()}", "DEBUG")
            if result and result.returncode == 0:
                self.log_device_result(serial, "Cài zip qua TWRP xong", "OK")
                time.sleep(2)
            else:
                err = result.stderr.strip() if result else "Không rõ"
                self.log_device_result(serial, f"Lỗi cài zip: {err}", "ERROR")
        except Exception as e:
            self.log_device_result(serial, f"Lỗi cài zip: {e}", "ERROR")

    def twrp_reboot_single(self, serial):
        try:
            cmd = [ADB, "-s", serial, "shell", "twrp reboot"]
            run_silent(cmd)
            self.log_device_result(serial, "Reboot xong", "OK")
        except Exception as e:
            self.log_device_result(serial, f"Lỗi reboot: {e}", "ERROR")

    def closeEvent(self, event):
        self.cleanup_processes()
        event.accept()
    def cleanup_processes(self):
        try:
            if os.name == "nt":
                os.system("taskkill /F /IM adb.exe /T >nul 2>&1")
                os.system("taskkill /F /IM fastboot.exe /T >nul 2>&1")
            else:
                os.system("pkill -f adb")
                os.system("pkill -f fastboot")
        except Exception as e:
            print("Cleanup error:", e)

if __name__ == "__main__":
    app = QApplication(sys.argv)
    w = AndroidMultiTool()
    w.show()
    sys.exit(app.exec_())

# Build File .EXE
# pyinstaller --onefile --windowed `
#   --icon=icon\OKPay.ico `
#   --hidden-import=PyQt5.sip `
#   --collect-submodules PyQt5 --collect-data PyQt5 `
#   `
#   --add-binary "adb.exe;." `
#   --add-binary "fastboot.exe;." `
#   --add-binary "AdbWinApi.dll;." `
#   --add-binary "AdbWinUsbApi.dll;." `
#   `
#   --add-data "icon;icon" `
#   `
#   --add-data "scrcpy\scrcpy-server;scrcpy" `
#   --add-binary "scrcpy\scrcpy.exe;scrcpy" `
#   --add-binary "scrcpy\SDL2.dll;scrcpy" `
#   --add-binary "scrcpy\libusb-1.0.dll;scrcpy" `
#   --add-binary "scrcpy\avcodec-61.dll;scrcpy" `
#   --add-binary "scrcpy\avformat-61.dll;scrcpy" `
#   --add-binary "scrcpy\avutil-59.dll;scrcpy" `
#   --add-binary "scrcpy\swresample-5.dll;scrcpy" `
#   `
#   --add-data "*.conf;." `
#   --add-data "*.txt;." `
#   `
#   --noconfirm --clean `
#   android_multi_tool_allin.py
