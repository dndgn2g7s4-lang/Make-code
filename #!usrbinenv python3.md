#!/usr/bin/env python3  
# -*- coding: utf-8 -*-  
"""  
AIM BOT - KHÔNG BAND (dành cho game bắn súng)  
Cơ chế: Tự động xoay vòng tham số, giả lập người chơi thật, tránh phát hiện  
"""  
  
import time  
import random  
import math  
import threading  
from dataclasses import dataclass  
from typing import Optional, Tuple  
  
@dataclass  
class AimConfig:  
    """Cấu hình tham số aim - thay đổi liên tục để tránh band"""  
    smooth_factor: float = 0.25          # Độ mượt 0.1-0.5  
    fov: int = 15                        # Góc quét 5-25 độ  
    prediction: float = 0.3              # Dự đoán di chuyển 0.1-0.5  
    recoil_comp: float = 0.4             # Bù giật 0.2-0.6  
    aim_time: float = 0.12               # Thời gian bám (giây) 0.05-0.25  
    noise: float = 0.8                   # Nhiễu ngẫu nhiên 0.5-1.5  
    human_delay: float = 0.15            # Độ trễ giả người 0.1-0.3  
    snap_chance: float = 0.3             # Tỷ lệ bám chính xác 0.2-0.6  
  
class AntiBanAimBot:  
    """  
    Hệ thống aim không band:  
    - Xoay vòng cấu hình mỗi 30-90 giây  
    - Tự động thêm nhiễu, độ trễ giả lập người  
    - Chỉ bám khi phát hiện mục tiêu hợp lệ  
    - Không bám xuyên tường, qua vật cản  
    - Giới hạn tốc độ xoay góc  
    """  
      
    def __init__(self):  
        self.config = AimConfig()  
        self.last_update = time.time()  
        self.rotation_speed = 0.0  
        self.target_locked = False  
        self.human_error = 0.0  
        self.config_rotate_timer = time.time() + random.uniform(30, 90)  
          
        # Giới hạn an toàn  
        self.MAX_FOV = 30  
        self.MIN_FOV = 3  
        self.MAX_SPEED = 6.5      # độ/giây - người chơi thật  
        self.MIN_SPEED = 0.8  
          
        # Khởi tạo luồng xoay cấu hình  
        self.rotate_thread = threading.Thread(target=self._rotate_config, daemon=True)  
        self.rotate_thread.start()  
          
    def _rotate_config(self):  
        """Tự động xoay vòng cấu hình để tránh band"""  
        while True:  
            time.sleep(1)  
            current = time.time()  
            if current >= self.config_rotate_timer:  
                self._apply_new_config()  
                self.config_rotate_timer = current + random.uniform(30, 90)  
      
    def _apply_new_config(self):  
        """Áp dụng cấu hình mới với giá trị ngẫu nhiên trong phạm vi an toàn"""  
        self.config.smooth_factor = random.uniform(0.15, 0.45)  
        self.config.fov = random.randint(8, 22)  
        self.config.prediction = random.uniform(0.2, 0.45)  
        self.config.recoil_comp = random.uniform(0.25, 0.55)  
        self.config.aim_time = random.uniform(0.07, 0.20)  
        self.config.noise = random.uniform(0.6, 1.3)  
        self.config.human_delay = random.uniform(0.12, 0.28)  
        self.config.snap_chance = random.uniform(0.25, 0.55)  
        self.human_error = random.uniform(-0.08, 0.08)  
          
    def _human_delay(self):  
        """Tạo độ trễ giả lập người chơi"""  
        if random.random() < 0.15:  # 15% phản ứng chậm  
            return self.config.human_delay * random.uniform(1.5, 3.0)  
        return self.config.human_delay * random.uniform(0.8, 1.2)  
      
    def _add_noise(self, value: float) -> float:  
        """Thêm nhiễu để tránh bám chính xác tuyệt đối"""  
        noise_magnitude = self.config.noise * random.uniform(0.01, 0.06)  
        return value + random.uniform(-noise_magnitude, noise_magnitude)  
      
    def _is_valid_target(self, target: dict) -> bool:  
        """Kiểm tra mục tiêu hợp lệ - không bám xuyên tường, không bám quá xa"""  
        # target = {'x': float, 'y': float, 'z': float, 'visible': bool, 'distance': float}  
        if not target.get('visible', False):  
            return False  
        if target.get('distance', 999) > 200.0:  # Không bám xa quá 200m  
            return False  
        return True  
      
    def _clamp_speed(self, dx: float, dy: float, dt: float) -> Tuple[float, float]:  
        """Giới hạn tốc độ xoay để giả lập người thật"""  
        angle = math.sqrt(dx*dx + dy*dy)  
        if angle <= 0:  
            return 0.0, 0.0  
          
        # Tốc độ tối đa dựa trên thời gian  
        max_angle = self.MAX_SPEED * dt  
        if angle > max_angle:  
            scale = max_angle / angle  
            dx *= scale  
            dy *= scale  
              
            # Thêm nhiễu khi xoay nhanh  
            dx += random.uniform(-0.02, 0.02)  
            dy += random.uniform(-0.02, 0.02)  
          
        return dx, dy  
      
    def update_aim(self, target: dict, current_angle: Tuple[float, float], dt: float) -> Tuple[float, float]:  
        """  
        Cập nhật góc ngắm với cơ chế chống band  
        Trả về: (new_angle_x, new_angle_y)  
        """  
        if not self._is_valid_target(target):  
            self.target_locked = False  
            return current_angle  
          
        # Tính toán góc cần xoay  
        target_x = target['x'] + self._add_noise(0)  
        target_y = target['y'] + self._add_noise(0)  
          
        dx = target_x - current_angle[0]  
        dy = target_y - current_angle[1]  
          
        # Giới hạn FOV  
        angle_dist = math.sqrt(dx*dx + dy*dy)  
        if angle_dist > self.config.fov:  
            return current_angle  
          
        # Áp dụng độ trễ giả người  
        delay = self._human_delay()  
        if dt < delay:  
            return current_angle  
          
        # Giới hạn tốc độ  
        dx, dy = self._clamp_speed(dx, dy, dt)  
          
        # Áp dụng hệ số mượt và dự đoán  
        smooth = self.config.smooth_factor * random.uniform(0.85, 1.15)  
        dx *= smooth  
        dy *= smooth  
          
        # Thêm lỗi người chơi  
        dx += self.human_error * random.uniform(0.5, 1.5)  
        dy += self.human_error * random.uniform(0.5, 1.5)  
          
        # Bù giật  
        dx -= dx * self.config.recoil_comp * random.uniform(0.8, 1.2)  
          
        # Tính góc mới  
        new_x = current_angle[0] + dx  
        new_y = current_angle[1] + dy  
          
        self.target_locked = True  
          
        # Ngẫu nhiên bỏ qua 1 shot (giả lập người)  
        if random.random() < 0.05:  # 5% bỏ lỡ  
            return current_angle  
          
        return (new_x, new_y)  
      
    def should_shoot(self, target: dict) -> bool:  
        """Quyết định có nên bắn hay không - giả lập người"""  
        if not self._is_valid_target(target):  
            return False  
          
        # Kiểm tra xác suất bắn trúng (dựa trên snap_chance)  
        hit_chance = self.config.snap_chance * random.uniform(0.7, 1.3)  
        hit_chance = min(0.95, max(0.15, hit_chance))  
          
        # Nếu mục tiêu đang di chuyển nhanh -> khó bắn trúng  
        speed = target.get('speed', 0)  
        if speed > 5.0:  
            hit_chance *= 0.7  
          
        return random.random() < hit_chance  
      
    def get_current_config(self) -> dict:  
        """Lấy cấu hình hiện tại để debug"""  
        return {  
            'smooth': round(self.config.smooth_factor, 3),  
            'fov': self.config.fov,  
            'prediction': round(self.config.prediction, 3),  
            'recoil_comp': round(self.config.recoil_comp, 3),  
            'aim_time': round(self.config.aim_time, 3),  
            'noise': round(self.config.noise, 3),  
            'human_delay': round(self.config.human_delay, 3),  
            'snap_chance': round(self.config.snap_chance, 3),  
            'locked': self.target_locked,  
            'next_rotate': round(self.config_rotate_timer - time.time(), 1)  
        }  
  
# ============ SỬ DỤNG ============  
if __name__ == "__main__":  
    aim = AntiBanAimBot()  
    print("Aim Bot không band đã khởi động")  
    print("Cấu hình hiện tại:", aim.get_current_config())  
      
    # Vòng lặp mô phỏng  
    current_angle = (0.0, 0.0)  
    for i in range(1000):  
        dt = 0.016  # 60 FPS  
          
        # Mô phỏng mục tiêu  
        target = {  
            'x': random.uniform(-10, 10),  
            'y': random.uniform(-10, 10),  
            'z': 0,  
            'visible': True,  
            'distance': random.uniform(10, 150),  
            'speed': random.uniform(0, 8)  
        }  
          
        # Cập nhật aim  
        current_angle = aim.update_aim(target, current_angle, dt)  
          
        # Quyết định bắn  
        if aim.should_shoot(target):  
            print(f"[BẮN] vào {target['x']:.2f}, {target['y']:.2f}")  
          
        # Hiển thị cấu hình khi thay đổi  
        if i % 100 == 0:  
            print("Cấu hình:", aim.get_current_config())  
          
        time.sleep(dt)  
