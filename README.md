# lab3.d


import os
import datetime
import mimetypes
from abc import ABC, abstractmethod
from collections import defaultdict
import threading
import time

class FileInfo(ABC):
    def __init__(self, filename):
        self.filename = filename

    @abstractmethod
    def get_info(self):
        pass

class ImageFileInfo(FileInfo):
    def get_info(self):
        file_path = os.path.join(folder_path, self.filename)
        if os.path.exists(file_path):
            with open(file_path, 'rb') as f:
                image_data = f.read()
            return len(image_data)
        else:
            return None

class TextFileInfo(FileInfo):
    def get_info(self):
        file_path = os.path.join(folder_path, self.filename)
        if os.path.exists(file_path):
            with open(file_path, 'r', encoding='utf-8') as f:
                lines = f.readlines()
            line_count = len(lines)
            word_count = sum(len(line.split()) for line in lines)
            character_count = sum(len(line) for line in lines)
            return line_count, word_count, character_count
        else:
            return None

class PythonFileInfo(FileInfo):
    def get_info(self):
        file_path = os.path.join(folder_path, self.filename)
        if os.path.exists(file_path):
            with open(file_path, 'r', encoding='utf-8') as f:
                lines = f.readlines()
            line_count = len(lines)
            class_count = sum(1 for line in lines if line.strip().startswith('class '))
            method_count = sum(1 for line in lines if line.strip().startswith('def '))
            return line_count, class_count, method_count
        else:
            return None

class JavaFileInfo(FileInfo):
    def get_info(self):
        file_path = os.path.join(folder_path, self.filename)
        if os.path.exists(file_path):
            with open(file_path, 'r', encoding='utf-8') as f:
                lines = f.readlines()
            line_count = len(lines)
            class_count = sum(1 for line in lines if line.strip().startswith('class '))
            method_count = sum(1 for line in lines if line.strip().startswith('public '))
            return line_count, class_count, method_count
        else:
            return None

class FileMonitor:
    def __init__(self, folder_path):
        self.folder_path = folder_path
        self.snapshot_time = None
        self.file_info = defaultdict(dict)
        self.instance = FileInfo
        self.file_type = None
        self.lock = threading.Lock()
        self.commit_time = None

    def get_file_instance(self, filename):
        file_path = os.path.join(self.folder_path, filename)
        if os.path.exists(file_path):
            file_type = mimetypes.guess_type(file_path)[0]
            self.file_type = file_type
            if file_type in ('image/png', 'image/jpeg'):
                instance = ImageFileInfo
            elif file_type == 'text/plain':
                instance = TextFileInfo
            elif file_type == 'text/x-python' or file_type == 'application/x-python-code':
                instance = PythonFileInfo
            elif file_type == 'text/x-java-source':
                instance = JavaFileInfo
            else:
                instance = FileInfo
            return instance
        else:
            return None

    def monitor_folder(self):
        with self.lock:
            current_files = set(os.listdir(self.folder_path))

            new_files = current_files - set(self.file_info.keys())
            for new_file in new_files:
                Instance = self.get_file_instance(new_file)
                if Instance:
                    self.file_info[new_file] = Instance(new_file).get_info()
                    print(f"{new_file} - New File") 

            for filename in current_files:
                if os.path.isfile(os.path.join(self.folder_path, filename)):
                    Instance = self.get_file_instance(filename)
                    if Instance:
                        self.file_info[filename] = Instance(filename).get_info()

    def info(self, filename):
        with self.lock:
            Instance = self.get_file_instance(filename)
            if Instance:
                file_instance = Instance(filename)
                file_info = file_instance.get_info()
                if file_info:
                    log_message = f"{datetime.datetime.now()} - Info requested for {filename}\n"
                    self.log_operation(log_message)
                    print(f"Name: {filename}")
                    print(f"Type: {self.file_type or file_instance.__class__.__name__}")
                    creation_time = os.path.getctime(os.path.join(self.folder_path, filename))
                    modification_time = os.path.getmtime(os.path.join(self.folder_path, filename))
                    print(f"Created: {datetime.datetime.fromtimestamp(creation_time)}")
                    print(f"Updated: {datetime.datetime.fromtimestamp(modification_time)}")

                    if isinstance(file_instance, ImageFileInfo):
                        print(f"Image Size: {file_info} bytes")
                    elif isinstance(file_instance, TextFileInfo):
                        print(f"Line Count: {file_info[0]}")
                        print(f"Word Count: {file_info[1]}")
                        print(f"Character Count: {file_info[2]}")
                    elif isinstance(file_instance, PythonFileInfo) or isinstance(file_instance, JavaFileInfo):
                        print(f"Line Count: {file_info[0]}")
                        print(f"Class Count: {file_info[1]}")
                        print(f"Method Count: {file_info[2]}")
                else:
                    print("File not found or does not exist.")
            else:
                print("File not found or does not exist.")

    def status(self):
        with self.lock:
            print("Status:")
            if not self.snapshot_time:
                print("No snapshot taken.")
                return

            current_files = set(os.listdir(self.folder_path))

            new_files = current_files - set(self.file_info.keys())
            for new_file in new_files:
                print(f"{new_file} - New File")

            deleted_files = set(self.file_info.keys()) - current_files
            for deleted_file in deleted_files:
                print(f"{deleted_file} - Deleted")

            print(f"Snapshot Time: {self.snapshot_time}")
            for filename, info in self.file_info.items():
                Instance = self.get_file_instance(filename)
                if Instance:
                    current_info = Instance(filename).get_info()
                    if current_info:
                        if current_info != info:
                            print(f"{filename} - Changed")
                        else:
                            print(f"{filename} - No Change")
                    else:
                        print(f"{filename} - Not found")

    def commit(self):
      with self.lock:
        self.snapshot_time = datetime.datetime.now()
        self.commit_time = self.snapshot_time.strftime("%Y-%m-%d_%H-%M-%S") 
        
        file_stats = "\nFile Statistics:\n"
        for filename, info in self.file_info.items():
            file_stats += f"{filename} - {info}\n"

        log_message = f"{self.commit_time} - Manual commit performed\n{file_stats}\n"
        print("Snapshot taken.")
        
        commit_filename = os.path.join(folder_path, f"commit_{self.commit_time}.txt")
        with open(commit_filename, "w") as commit_file:
            commit_file.write(log_message)


def scheduled_detection(monitor):
    while True:
        monitor.monitor_folder()
        monitor.status()
        time.sleep(5)

if __name__ == "__main__":
    folder_path = r'C:\Users\vasil\OneDrive\Desktop\Catalin OOP\SibaevVasile_Lab3_PAPP_OOP'
    monitor = FileMonitor(folder_path)
    # Start a separate thread for scheduled detection
    detection_thread = threading.Thread(target=scheduled_detection, args=(monitor,))
    detection_thread.start()

    while True:
        command = input(
            "Enter command (info <filename>/commit/status/exit): ").strip().split()
        if command[0] == 'info' and len(command) == 2:
            monitor.info(command[1])
        elif command[0] == 'commit':
            monitor.commit()
        elif command[0] == 'status':
            monitor.status()
        elif command[0] == 'exit':
            break
        else:
            print("Invalid command. Please try again.")
