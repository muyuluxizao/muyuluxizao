import os
import cv2
import tkinter as tk
from tkinter import filedialog, messagebox
from tkinter import filedialog, messagebox, ttk
from moviepy.editor import VideoFileClip, concatenate_videoclips
from skimage.metrics import structural_similarity as ssim
import concurrent.futures

class VideoEditor:
    def __init__(self):
        self.window = tk.Tk()
        print("Creating VideoEditor instance...")
        self.window.title("视频剪辑软件")
        self.shared_video = None
        self.shared_compare = None
        self.concatenate_option = tk.StringVar()
        self.concatenate_option.set("auto")  # 默认为自动拼接
        self.similarity_threshold = 0.9  # 相似性阈值

    def run(self):
        self.initialize_gui()

    def handle_auto_clip_button(self):
        if self.shared_video and self.shared_compare:
            # 场景检测
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)

            # 相似片段检测
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            # 调用 auto_clip 方法并传递变量
            self.auto_clip(scenes_video, scenes_compare, similar_segments)
        else:
            self.info_label.config(text="请选择原视频和对比视频")

    def auto_clip(self, scenes_video, scenes_compare, similar_segments):
        # 拼接场景和相似片段
        all_segments = scenes_video + scenes_compare + similar_segments
        all_segments.sort(key=lambda x: x[0])  # 按照时间戳排序

        if self.concatenate_option.get() == "auto":
            # 自动拼接视频
            self.run_background_task(self.background_auto_clip, all_segments, callback=self.handle_auto_clip_complete)
        elif self.concatenate_option.get() == "manual":
            # 手动拼接的逻辑
            pass
        else:
            self.info_label.config(text="请选择原视频和对比视频")

    def background_auto_clip(self, all_segments):
        return self.concatenate_segments(self.shared_video, all_segments)

    def handle_auto_clip_complete(self, result):
        # 处理自动剪辑完成后的操作
        if result:
            self.output_path.set(result)
            self.update_text(f"视频剪辑成功，自动拼接，保存在本地文件：{result}\n")
            self.show_info("提示", "视频剪辑成功，自动拼接")

    def concatenate_segments(self, video_clip, segments):
        # 拼接视频片段的逻辑
        concatenated_clip = concatenate_videoclips([video_clip.subclip(start, end) for start, end in segments])

        # 定义拼接后的视频保存路径
        output_path = filedialog.asksaveasfilename(defaultextension=".mp4", filetypes=[("MP4 files", "*.mp4")])

        if output_path:
            # 保存拼接后的视频
            concatenated_clip.write_videofile(output_path, codec="libx264", audio_codec="aac")

        return output_path

    def run_background_task(self, task_func, *args, callback=None):
        def background_task():
            result = task_func(*args)
            if callback:
                self.window.after(0, callback, result)

        with concurrent.futures.ThreadPoolExecutor() as executor:
            executor.submit(background_task)

    # 场景检测、相似片段检测等方法...


    def import_original(self):
        original_file_path = filedialog.askopenfilename(title="选择原视频文件", filetypes=[("视频文件", "*.mp4;*.avi")])

        if original_file_path:
            self.shared_video = VideoFileClip(original_file_path)
            self.show_info("提示", "原视频导入成功")

    def import_compare(self):
        compare_file_path = filedialog.askopenfilename(title="选择对比视频文件", filetypes=[("视频文件", "*.mp4;*.avi")])

        if compare_file_path:
            self.shared_compare = VideoFileClip(compare_file_path)
            self.show_info("提示", "对比视频导入成功")

    def scene_detection(self, video_clip):
        scenes = []
        prev_frame = None

        for i, (frame, _) in enumerate(video_clip.iter_frames(fps=video_clip.fps, with_times=True)):
            gray_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)

            if prev_frame is not None:
                diff = cv2.absdiff(prev_frame, gray_frame)
                _, threshold = cv2.threshold(diff, 30, 255, cv2.THRESH_BINARY)
                contours, _ = cv2.findContours(threshold, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)

                if len(contours) > 0:
                    scenes.append((i / video_clip.fps, (i + 1) / video_clip.fps))

            prev_frame = gray_frame

        return scenes

    def detect_similar_segments(self, video_clip1, video_clip2):
        similar_segments = []
        for i, (frame1, frame2) in enumerate(zip(video_clip1.iter_frames(fps=video_clip1.fps),
                                                 video_clip2.iter_frames(fps=video_clip2.fps))):
            gray_frame1 = cv2.cvtColor(frame1, cv2.COLOR_BGR2GRAY)
            gray_frame2 = cv2.cvtColor(frame2, cv2.COLOR_BGR2GRAY)

            similarity = ssim(gray_frame1, gray_frame2)

            if similarity > self.similarity_threshold:
                similar_segments.append((i / video_clip1.fps, (i + 1) / video_clip1.fps))

        return similar_segments

    def clip_segments(self, video_clip, segments):
        clipped_segments = []

        for start, end in segments:
            subclip = video_clip.subclip(start, end)
            clipped_segments.append(subclip)

        return clipped_segments

    def concatenate_segments(self, video_clip, segments):
        concatenated_clip = concatenate_videoclips(segments)
        output_path = self.choose_output_file("保存拼接后的视频", [("视频文件", "*.mp4")])

        if output_path:
            concatenated_clip.write_videofile(output_path, codec="libx264", audio_codec="aac")
            return output_path
        else:
            self.show_error("错误", "未提供输出路径")

    def export_segments_separately(self, video_clip, segments, output_folder='E:/output'):
        if not os.path.exists(output_folder):
            os.makedirs(output_folder)

        for i, segment in enumerate(segments):
            start_time, end_time = segment
            subclip = video_clip.subclip(start_time, end_time)
            output_path = os.path.join(output_folder, f'segment_{i + 1}.mp4')
            subclip.write_videofile(output_path, codec='libx264', audio_codec='aac')

    def choose_output_file(self, title, filetypes):
        return filedialog.asksaveasfilename(title=title, filetypes=filetypes, defaultextension=filetypes[0][1])



    def handle_concatenate_button(self):
        if self.shared_video and self.shared_compare:
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])

            self.concatenate_segments(self.shared_video, all_segments)
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_export_segments_button(self):
        if self.shared_video and self.shared_compare:
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])

            self.export_segments_separately(self.shared_video, all_segments)
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_auto_clip_button(self):
        if self.shared_video and self.shared_compare:
            self.run_background_task(self.background_auto_clip, callback=self.handle_auto_clip_complete)
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_auto_clip_complete(self, result):
        if result:
            self.output_path.set(result)
            self.update_text(f"视频剪辑成功，自动拼接，保存在本地文件：{result}\n")
            self.show_info("提示", "视频剪辑成功，自动拼接")

    def background_auto_clip(self):
        scenes_video = self.scene_detection(self.shared_video)
        scenes_compare = self.scene_detection(self.shared_compare)
        similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

        all_segments = scenes_video + scenes_compare + similar_segments
        all_segments.sort(key=lambda x: x[0])

        if self.concatenate_option.get() == "auto":
            return self.concatenate_segments(self.shared_video, all_segments)
        elif self.concatenate_option.get() == "manual":
            # 手动拼接的逻辑
            pass
        return None

    def export_video(self):
        if self.output_path.get():
            messagebox.showinfo("导出", f"视频已成功导出至：{self.output_path.get()}")
        else:
            messagebox.showerror("错误", "请先进行自动剪辑并保存视频")

    def clear_text(self):
        self.text_box.delete(1.0, tk.END)

    def update_text(self, message):
        self.text_box.insert(tk.END, message)
        self.text_box.see(tk.END)

    def show_info(self, title, message):
        messagebox.showinfo(title, message)

    def show_error(self, title, message):
        messagebox.showerror(title, message)

    def run_background_task(self, task_func, callback=None):
        def background_task():
            result = task_func()
            if callback:
                self.window.after(0, callback, result)

        with concurrent.futures.ThreadPoolExecutor() as executor:
            executor.submit(background_task)


    def import_original(self):
        original_file_path = filedialog.askopenfilename(title="选择原视频文件", filetypes=[("视频文件", "*.mp4;*.avi")])

        if original_file_path:
            self.shared_video = VideoFileClip(original_file_path)
            self.show_info("提示", "原视频导入成功")

    def import_compare(self):
        compare_file_path = filedialog.askopenfilename(title="选择对比视频文件", filetypes=[("视频文件", "*.mp4;*.avi")])

        if compare_file_path:
            self.shared_compare = VideoFileClip(compare_file_path)
            self.show_info("提示", "对比视频导入成功")

    def auto_clip(self):
        if self.shared_video and self.shared_compare:
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])

            if self.concatenate_option.get() == "auto":
                self.run_background_task(self.background_auto_clip, callback=self.handle_auto_clip_complete)
            elif self.concatenate_option.get() == "manual":
                # 在这里实现手动拼接的逻辑
                pass
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_concatenate_button(self):
        if self.shared_video and self.shared_compare:
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])

            self.concatenate_segments(self.shared_video, all_segments)
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_export_segments_button(self):
        if self.shared_video and self.shared_compare:
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])

            self.export_segments_separately(self.shared_video, all_segments)
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_auto_clip_complete(self, result):
        if result:
            self.output_path.set(result)
            self.update_text(f"视频剪辑成功，自动拼接，保存在本地文件：{result}\n")
            self.show_info("提示", "视频剪辑成功，自动拼接")

    def handle_manual_clip_button(self):
        if self.shared_video and self.shared_compare:
            # 在这里实现手动拼接的逻辑
            pass
        else:
            self.show_error("错误", "请选择原视频和对比视频")



    def handle_export_video_button(self):
        if self.output_path.get():
            self.export_video()
        else:
            self.show_error("错误", "请先进行自动剪辑并保存视频")

    def clear_text(self):
        self.text_box.delete(1.0, tk.END)

    def update_text(self, message):
        self.text_box.insert(tk.END, message)
        self.text_box.see(tk.END)

    def show_info(self, title, message):
        messagebox.showinfo(title, message)

    def show_error(self, title, message):
        messagebox.showerror(title, message)

    def handle_concatenate_button(self):
        if self.shared_video and self.shared_compare:
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])

            self.run_background_task(self.background_concatenate, callback=self.handle_concatenate_complete)
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_export_segments_button(self):
        if self.shared_video and self.shared_compare:
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])

            self.run_background_task(self.background_export_segments, callback=self.handle_export_segments_complete)
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_auto_clip_button(self):
        if self.shared_video and self.shared_compare:
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])

            self.run_background_task(self.background_auto_clip, callback=self.handle_auto_clip_complete)
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_manual_clip_button(self):
        if self.shared_video and self.shared_compare:
            # 在这里实现手动拼接的逻辑
            pass
        else:
            self.show_error("错误", "请选择原视频和对比视频")


    def handle_manual_clip_button(self):
        if self.shared_video and self.shared_compare:
            # 在这里实现手动拼接的逻辑
            pass
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_export_video_button(self):
        if self.output_path.get():
            self.export_video()
        else:
            self.show_error("错误", "请先进行自动剪辑并保存视频")

    def clear_text(self):
        self.text_box.delete(1.0, tk.END)

    def update_text(self, message):
        self.text_box.insert(tk.END, message)
        self.text_box.see(tk.END)

    def show_info(self, title, message):
        messagebox.showinfo(title, message)

    def show_error(self, title, message):
        messagebox.showerror(title, message)

    def handle_concatenate_button(self):
        if self.shared_video and self.shared_compare:
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])

            self.run_background_task(self.background_concatenate, callback=self.handle_concatenate_complete)
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_export_segments_button(self):
        if self.shared_video and self.shared_compare:
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])

            self.run_background_task(self.background_export_segments, callback=self.handle_export_segments_complete)
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_auto_clip_button(self):
        if self.shared_video and self.shared_compare:
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])

            self.run_background_task(self.background_auto_clip, callback=self.handle_auto_clip_complete)
        else:
            self.show_error("错误", "请选择原视频和对比视频")

    def handle_manual_clip_button(self):
        if self.shared_video and self.shared_compare:
            # 在这里实现手动拼接的逻辑
            pass
        else:
            self.show_error("错误", "请选择原视频和对比视频")


    def background_concatenate(self):
        output_path = self.concatenate_segments(self.shared_video, self.all_segments)
        return output_path

    def handle_concatenate_complete(self, result):
        # 处理拼接完成后的操作
        if result:
            self.output_path.set(result)
            self.update_text(f"视频拼接成功，保存在本地文件：{result}\n")
            self.show_info("提示", "视频拼接成功")
        else:
            self.show_error("错误", "视频拼接失败")

    def background_export_segments(self):
        self.export_segments_separately(self.shared_video, self.all_segments)

    def handle_export_segments_complete(self, result):
        # 处理导出片段完成后的操作
        if result:
            self.show_info("提示", "视频片段导出成功")
        else:
            self.show_error("错误", "视频片段导出失败")

    def background_auto_clip(self):
        output_path = self.concatenate_segments(self.shared_video, self.all_segments)
        return output_path

    def handle_auto_clip_complete(self, result):
        # 处理自动剪辑完成后的操作
        if result:
            self.output_path.set(result)
            self.update_text(f"视频剪辑成功，自动拼接，保存在本地文件：{result}\n")
            self.show_info("提示", "视频剪辑成功，自动拼接")
        else:
            self.show_error("错误", "自动剪辑失败，请重试")

    def run_background_task(self, task_func, callback=None):
        def background_task():
            result = task_func()
            if callback:
                self.window.after(0, callback, result)

        with concurrent.futures.ThreadPoolExecutor() as executor:
            executor.submit(background_task)

    def concatenate_segments(self, video_clip, segments):
        concatenated_clip = concatenate_videoclips([video_clip.subclip(start, end) for start, end in segments])
        output_path = self.choose_output_file("保存拼接后的视频", [("视频文件", "*.mp4")])

        if output_path:
            concatenated_clip.write_videofile(output_path, codec="libx264", audio_codec="aac")
            return output_path
        else:
            self.show_error("错误", "未提供输出路径")
            return None

    def export_segments_separately(self, video_clip, segments):
        output_folder = filedialog.askdirectory(title="选择导出文件夹")

        if output_folder:
            for i, (start, end) in enumerate(segments):
                subclip = video_clip.subclip(start, end)
                output_path = os.path.join(output_folder, f'segment_{i + 1}.mp4')
                subclip.write_videofile(output_path, codec='libx264', audio_codec='aac')
        else:
            self.show_error("错误", "未提供输出文件夹")


    def import_original(self):
        original_file_path = filedialog.askopenfilename(title="选择原视频文件", filetypes=[("视频文件", "*.mp4;*.avi")])

        if original_file_path:
            # 用用户选择的原视频文件名和路径创建一个 VideoFileClip 对象
            self.shared_video = VideoFileClip(original_file_path)

            # 使用 tkinter.messagebox 显示提示信息
            self.show_info("提示", "原视频导入成功")

    def import_compare(self):
        compare_file_path = filedialog.askopenfilename(title="选择对比视频文件", filetypes=[("视频文件", "*.mp4;*.avi")])

        if compare_file_path:
            # 用用户选择的对比视频文件名和路径创建一个 VideoFileClip 对象
            self.shared_compare = VideoFileClip(compare_file_path)

            # 使用 tkinter.messagebox 显示提示信息
            self.show_info("提示", "对比视频导入成功")


class VideoEditor:
    def __init__(self):
        self.window = tk.Tk()
        self.window.title("视频剪辑软件")
        self.shared_video = None
        self.shared_compare = None
        self.concatenate_option = tk.StringVar()
        self.concatenate_option.set("auto")  # 默认为自动拼接
        self.similarity_threshold = 0.9  # 相似性阈值

        self.progress_var = tk.DoubleVar()
        self.progress_var.set(0.0)

    def run(self):
        self.initialize_gui()

def initialize_gui(self):
    print("Initializing GUI...")  # 添加这一行
    # 创建 GUI 界面的其余部分...

    # 创建导入原视频按钮
    button_import_original = tk.Button(self.window, text="导入原视频", command=self.import_original)
    button_import_original.grid(row=1, column=0, padx=10, pady=5)

    # 创建导入对比视频按钮
    button_import_compare = tk.Button(self.window, text="导入对比视频", command=self.import_compare)
    button_import_compare.grid(row=1, column=1, padx=10, pady=5)

    # 创建“一键自动剪辑”按钮
    button_auto_clip = tk.Button(self.window, text="一键自动剪辑", command=self.handle_auto_clip_button)
    button_auto_clip.grid(row=2, column=0, padx=10, pady=5)

    # 创建导出按钮
    button_export_video = tk.Button(self.window, text="导出视频", command=self.export_video)
    button_export_video.grid(row=2, column=1, padx=10, pady=5)

    # ... (其他按钮的创建)

    # 创建进度条
    self.progress_bar = ttk.Progressbar(self.window, orient='horizontal', length=200, mode='determinate', variable=self.progress_var)
    self.progress_bar.grid(row=5, column=0, columnspan=2, padx=10, pady=5)
    print("GUI Initialization Complete")  # 添加这一行
# ... (其他方法和函数)

def export_video(self):
    if self.output_path.get():
        messagebox.showinfo("导出", f"视频已成功导出至：{self.output_path.get()}")
    else:
        messagebox.showerror("错误", "请先进行自动剪辑并保存视频")


    def auto_clip(self):
        if self.shared_video and self.shared_compare:
            # 场景检测
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)

            # 相似片段检测
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            # 拼接场景和相似片段
            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])  # 按照时间戳排序

            if self.concatenate_option.get() == "auto":
                # 自动拼接视频
                self.run_background_task(self.background_auto_clip, callback=self.handle_auto_clip_complete)
            elif self.concatenate_option.get() == "manual":
                # 手动拼接的逻辑
                pass
        else:
            self.info_label.config(text="请选择原视频和对比视频")

    def handle_auto_clip_complete(self, result):
        # 处理自动剪辑完成后的操作
        if result:
            self.output_path.set(result)
            self.update_text(f"视频剪辑成功，自动拼接，保存在本地文件：{result}\n")
            self.show_info("提示", "视频剪辑成功，自动拼接")

def auto_clip(self, scenes_video, scenes_compare, similar_segments):
    # 拼接场景和相似片段
    all_segments = scenes_video + scenes_compare + similar_segments
    all_segments.sort(key=lambda x: x[0])  # 按照时间戳排序

    if self.concatenate_option.get() == "auto":
        # 自动拼接视频
        self.run_background_task(self.background_auto_clip, all_segments, callback=self.handle_auto_clip_complete)
    elif self.concatenate_option.get() == "manual":
        # 手动拼接的逻辑
        pass
    else:
        self.info_label.config(text="请选择原视频和对比视频")

def background_auto_clip(self, all_segments):
    total_segments = len(all_segments)
    processed_segments = 0

    for i, (start, end) in enumerate(all_segments):
        self.progress_var.set((processed_segments / total_segments) * 100)

        # 拼接视频片段的逻辑
        subclip = self.shared_video.subclip(start, end)
        output_path = self.choose_output_file("保存拼接后的视频", [("视频文件", "*.mp4")])

        if output_path:
            subclip.write_videofile(output_path, codec="libx264", audio_codec="aac")
            processed_segments += 1

    return output_path



if __name__ == '__main__':
    editor = VideoEditor()
    editor.run()
from moviepy.editor import VideoFileClip, concatenate_videoclips
import tkinter as tk
from tkinter import filedialog, messagebox
import cv2
import os

class VideoEditor:
    def __init__(self):
        self.window = tk.Tk()
        self.window.title("视频剪辑软件")
        self.shared_video = None
        self.shared_compare = None
        self.concatenate_option = tk.StringVar()
        self.concatenate_option.set("auto")  # 默认为自动拼接
        self.similarity_threshold = 0.9  # 相似性阈值

    def run(self):
        self.initialize_gui()

    def initialize_gui(self):
        # 创建 GUI 界面的其余部分...
        pass

    # ... (之前的代码)

    def handle_auto_clip_button(self):
        self.auto_clip()

    def auto_clip(self):
        if self.shared_video and self.shared_compare:
            # 场景检测
            scenes_video = self.scene_detection(self.shared_video)
            scenes_compare = self.scene_detection(self.shared_compare)

            # 相似片段检测
            similar_segments = self.detect_similar_segments(self.shared_video, self.shared_compare)

            # 拼接场景和相似片段
            all_segments = scenes_video + scenes_compare + similar_segments
            all_segments.sort(key=lambda x: x[0])  # 按照时间戳排序

            if self.concatenate_option.get() == "auto":
                # 自动拼接视频
                output_path = self.concatenate_segments(self.shared_video, all_segments)
                self.update_text(f"视频剪辑成功，自动拼接，保存在本地文件：{output_path}\n")
                self.show_info("提示", "视频剪辑成功，自动拼接")
            elif self.concatenate_option.get() == "manual":
                # 手动拼接的逻辑
                pass
        else:
            self.info_label.config(text="请选择原视频和对比视频")

    def concatenate_segments(self, video_clip, segments):
        # 拼接视频片段的逻辑
        concatenated_clip = concatenate_videoclips([video_clip.subclip(start, end) for start, end in segments])

        # 定义拼接后的视频保存路径
        output_path = filedialog.asksaveasfilename(defaultextension=".mp4", filetypes=[("MP4 files", "*.mp4")])

        if output_path:
            # 保存拼接后的视频
            concatenated_clip.write_videofile(output_path, codec="libx264", audio_codec="aac")

        return output_path

    def export_video(self):
        if self.output_path.get():
            messagebox.showinfo("导出", f"视频已成功导出至：{self.output_path.get()}")
        else:
            messagebox.showerror("错误", "请先进行自动剪辑并保存视频")

    def clear_text(self):
        self.text_box.delete(1.0, tk.END)

    def update_text(self, message):
        self.text_box.insert(tk.END, message)
        self.text_box.see(tk.END)

    def show_info(self, title, message):
        messagebox.showinfo(title, message)

    def show_error(self, title, message):
        messagebox.showerror(title, message)

if __name__ == '__main__':
    editor = VideoEditor()
    editor.run()
