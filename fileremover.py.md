import os
import shutil
import hashlib
import magic
import argparse
from datetime import datetime

class FileManager:
    def __init__(self, verbose=False, backup=False, log_file=None):
        self.verbose = verbose
        self.backup = backup
        self.log_file = log_file
        self.backup_dir = "file_manager_backup"
        
        if self.backup and not os.path.exists(self.backup_dir):
            os.makedirs(self.backup_dir)
        
    def log(self, message):
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        log_message = f"[{timestamp}] {message}"
        
        if self.verbose:
            print(log_message)
            
        if self.log_file:
            with open(self.log_file, "a") as f:
                f.write(log_message + "\n")
    
    def get_file_info(self, path):
        """Get detailed information about a file"""
        info = {
            'path': path,
            'size': os.path.getsize(path),
            'modified': datetime.fromtimestamp(os.path.getmtime(path)),
            'created': datetime.fromtimestamp(os.path.getctime(path)),
            'is_file': os.path.isfile(path),
            'is_dir': os.path.isdir(path),
            'md5': self.get_file_hash(path, 'md5'),
            'sha1': self.get_file_hash(path, 'sha1'),
            'mime_type': magic.from_file(path, mime=True)
        }
        return info
    
    def get_file_hash(self, path, algorithm='md5'):
        """Calculate file hash using specified algorithm"""
        hash_func = getattr(hashlib, algorithm)()
        with open(path, 'rb') as f:
            for chunk in iter(lambda: f.read(4096), b""):
                hash_func.update(chunk)
        return hash_func.hexdigest()
    
    def backup_file(self, path):
        """Create backup of a file"""
        if not self.backup:
            return
            
        backup_path = os.path.join(self.backup_dir, os.path.basename(path))
        if os.path.exists(backup_path):
            backup_path = f"{backup_path}.{int(datetime.now().timestamp())}"
            
        shutil.copy2(path, backup_path)
        self.log(f"Backed up {path} to {backup_path}")
    
    def delete_file(self, path):
        """Securely delete a file"""
        self.backup_file(path)
        os.remove(path)
        self.log(f"Deleted file: {path}")
    
    def delete_folder(self, path):
        """Delete a folder recursively"""
        self.backup_file(path)
        shutil.rmtree(path)
        self.log(f"Deleted folder: {path}")
    
    def clean_directory(self, path='.', exclude=None, file_patterns=None, folder_patterns=None):
        """
        Clean a directory based on various criteria
        :param path: Directory to clean
        :param exclude: List of files/folders to exclude
        :param file_patterns: List of file patterns to match (e.g., ['*.tmp', '*.log'])
        :param folder_patterns: List of folder patterns to match
        """
        if exclude is None:
            exclude = [os.path.basename(__file__), self.backup_dir]
        
        for item in os.listdir(path):
            item_path = os.path.join(path, item)
            
            if item in exclude:
                self.log(f"Skipping excluded item: {item}")
                continue
                
            if os.path.isfile(item_path):
                if file_patterns and not any(item.endswith(pattern.replace('*', '')) for pattern in file_patterns):
                    continue
                self.delete_file(item_path)
            elif os.path.isdir(item_path):
                if folder_patterns and not any(item.endswith(pattern.replace('*', '')) for pattern in folder_patterns):
                    continue
                self.delete_folder(item_path)

def main():
    parser = argparse.ArgumentParser(description="Advanced CTF File Manipulation Tool")
    parser.add_argument('-d', '--directory', default='.', help='Directory to operate on')
    parser.add_argument('-e', '--exclude', nargs='+', help='Files/folders to exclude')
    parser.add_argument('-fp', '--file-patterns', nargs='+', help='File patterns to match (e.g., *.tmp)')
    parser.add_argument('-dp', '--dir-patterns', nargs='+', help='Directory patterns to match')
    parser.add_argument('-b', '--backup', action='store_true', help='Enable backup before deletion')
    parser.add_argument('-v', '--verbose', action='store_true', help='Verbose output')
    parser.add_argument('-l', '--log', help='Log file path')
    parser.add_argument('--info', help='Get info about a specific file')
    parser.add_argument('--hash', choices=['md5', 'sha1', 'sha256'], help='Calculate hash of a file')
    
    args = parser.parse_args()
    
    fm = FileManager(verbose=args.verbose, backup=args.backup, log_file=args.log)
    
    if args.info:
        info = fm.get_file_info(args.info)
        for k, v in info.items():
            print(f"{k}: {v}")
        return
    
    if args.hash and args.info:
        print(f"{args.hash.upper()} hash: {fm.get_file_hash(args.info, args.hash)}")
        return
    
    fm.clean_directory(
        path=args.directory,
        exclude=args.exclude,
        file_patterns=args.file_patterns,
        folder_patterns=args.dir_patterns
    )

if __name__ == "__main__":
    try:
        main()
    except Exception as e:
        print(f"Error: {e}")

