<?php
ini_set('max_execution_time', 300);
ini_set('memory_limit', '256M');

class ShellScanner {
    private $suspiciousFiles = [
        '.php', '.phtml', '.php3', '.php4', '.php5', '.php7', '.pht', '.phps',
        '.cgi', '.pl', '.py', '.jsp', '.asp', '.aspx', '.sh', '.bash'
    ];

    private $suspiciousNames = [
        'shell', 'backdoor', 'hack', 'security', 'sec', 'injection', 'wso',
        'cmd', 'root', 'upload', 'webadmin', 'admin', 'alfa', 'c99', 'r57',
        'b374k', 'c100', 'marijuana', 'predator', 'sad', 'spy', 'worm', 'dra'
    ];

    private $patterns = [
        'eval\s*\(' => 'Eval kullanımı',
        'base64_decode' => 'Base64 kod',
        'system\s*\(' => 'Sistem komutu',
        'exec\s*\(' => 'Exec komutu',
        'shell_exec' => 'Shell komutu',
        'passthru' => 'Passthru kullanımı',
        '\$_POST\s*\[.*\]\s*\(' => 'POST ile kod çalıştırma',
        '\$_GET\s*\[.*\]\s*\(' => 'GET ile kod çalıştırma',
        'move_uploaded_file' => 'Dosya yükleme',
        'file_get_contents' => 'Dosya okuma',
        'file_put_contents' => 'Dosya yazma',
        'str_rot13' => 'ROT13 şifreleme',
        'gzinflate' => 'GZIP çözme',
        'gzuncompress' => 'GZIP çözme',
        'error_reporting\(0\)' => 'Hata gizleme'
    ];

    private $count = 0;
    private $threats = 0;
    private $startTime;
    private $foundThreats = [];

    private function showHeader() {
        echo '<!DOCTYPE html>
        <html>
        <head>
            <meta charset="UTF-8">
            <title>Web Shell Tarayıcı by tron, @cyber0x8</title>
            <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
            <style>
                body { font-family: Arial, sans-serif; margin: 20px; background: #f0f2f5; }
                .container { max-width: 1200px; margin: 0 auto; }
                .header { background: #1a237e; color: white; padding: 20px; border-radius: 8px; margin-bottom: 20px; }
                .progress { position: sticky; top: 20px; background: #fff; padding: 20px; border-radius: 8px; 
                           margin-bottom: 20px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); z-index: 100; }
                .threat { background: #ff5252; color: white; padding: 15px; border-radius: 8px; margin-bottom: 10px; 
                         animation: slideIn 0.3s ease-out; }
                @keyframes slideIn { from { opacity: 0; transform: translateY(20px); } 
                                   to { opacity: 1; transform: translateY(0); } }
                .threat-high { background: #d32f2f; }
                .threat-medium { background: #f44336; }
                .threat-low { background: #ff5252; }
                .threat-info { background: #fff; padding: 15px; border-radius: 4px; margin-top: 10px; color: #333; }
                .matches { background: #ffebee; padding: 10px; border-radius: 4px; margin-top: 10px; }
                .match-item { color: #c62828; margin: 5px 0; }
                .delete-btn { background: #d32f2f; color: white; padding: 5px 10px; border: none; 
                             border-radius: 4px; cursor: pointer; margin-right: 5px; }
                .delete-btn:hover { background: #b71c1c; }
                .view-btn { background: #1565c0; color: white; padding: 5px 10px; border: none; 
                           border-radius: 4px; cursor: pointer; }
                .view-btn:hover { background: #0d47a1; }
                .content-modal { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%;
                                background: rgba(0,0,0,0.7); z-index: 1000; }
                .modal-box { background: #fff; margin: 5% auto; padding: 20px; width: 90%; max-width: 800px;
                            border-radius: 8px; max-height: 80vh; overflow-y: auto; }
                .stats { display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: 10px;
                        background: #fff; padding: 15px; border-radius: 8px; margin: 20px 0; }
                .stat-item { text-align: center; padding: 10px; background: #f5f5f5; border-radius: 4px; }
                .icon { margin-right: 5px; }
            </style>
            <script>
                function findAndRemoveThreats(filePath) {
                    const threats = document.querySelectorAll(".threat");
                    threats.forEach(threat => {
                        const pathElement = threat.querySelector(".threat-info p");
                        if (pathElement && pathElement.textContent.includes(filePath)) {
                            threat.remove();
                        }
                    });
                }

                function updateThreatCount(delta) {
                    const statsElement = document.querySelector(".stat-item:nth-child(2)");
                    if (statsElement) {
                        const currentText = statsElement.textContent;
                        const currentCount = parseInt(currentText.match(/\d+/)[0]) || 0;
                        const newCount = Math.max(0, currentCount + delta);
                        
                        statsElement.innerHTML = `
                            <i class="fas fa-exclamation-triangle icon"></i>
                            <strong>Tehditler:</strong> ${newCount}
                        `;
                    }
                }

                async function deleteFile(filePath, element) {
                    if (!confirm("Bu dosyayı silmek istediğinizden emin misiniz?")) return;
                    
                    try {
                        const response = await fetch("", {
                            method: "POST",
                            headers: {"Content-Type": "application/x-www-form-urlencoded"},
                            body: "delete_file=" + encodeURIComponent(filePath)
                        });
                        
                        const result = await response.json();
                        if (result.success) {
                            element.closest(".threat").remove();
                            updateThreatCount(-1);
                        } else {
                            alert("Hata: " + result.message);
                        }
                    } catch (error) {
                        alert("Bir hata oluştu: " + error);
                    }
                }

                async function deleteByName(filePath, rootDir) {
    if (!confirm("Bu dosya ile aynı isme sahip tüm dosyaları silmek istediğinizden emin misiniz?")) return;
    
    try {
        const response = await fetch("", {
            method: "POST",
            headers: {"Content-Type": "application/x-www-form-urlencoded"},
            body: "delete_by_name=" + encodeURIComponent(filePath) + 
                  "&root_dir=" + encodeURIComponent(rootDir)
        });
        
        const result = await response.json();
        if (result.success) {
            result.deletedFiles.forEach(path => {
                const threats = document.querySelectorAll(".threat");
                threats.forEach(threat => {
                    if (threat.textContent.includes(path)) {
                        threat.remove();
                    }
                });
            });
            
            updateThreatCount(-result.deletedFiles.length);
            
            alert(result.message);
        } else {
            alert("Hata: " + result.message);
        }
    } catch (error) {
        alert("Bir hata oluştu: " + error);
    }
}

                async function viewContent(filePath) {
                    try {
                        const response = await fetch("", {
                            method: "POST",
                            headers: {"Content-Type": "application/x-www-form-urlencoded"},
                            body: "view_content=" + encodeURIComponent(filePath)
                        });
                        
                        const result = await response.json();
                        if (result.success) {
                            document.getElementById("file-content").textContent = result.content;
                            document.getElementById("content-modal").style.display = "block";
                        } else {
                            alert("Hata: " + result.message);
                        }
                    } catch (error) {
                        alert("Bir hata oluştu: " + error);
                    }
                }

                function closeModal() {
                    document.getElementById("content-modal").style.display = "none";
                }

                document.addEventListener("keydown", function(event) {
                    if (event.key === "Escape") {
                        closeModal();
                    }
                });
            </script>
        </head>
        <body>
            <div class="container">
                <div class="header">
                    <h1><i class="fas fa-search icon"></i> Web Shell Tarayıcı by tron, @cyber0x8</h1>
                    <p>Başlangıç: ' . date('Y-m-d H:i:s') . '</p>
                </div>
                <div id="content-modal" class="content-modal">
                    <div class="modal-box">
                        <button onclick="closeModal()" style="float:right;padding:5px 10px;">Kapat</button>
                        <h3>Dosya İçeriği</h3>
                        <pre id="file-content" style="background:#f5f5f5;padding:15px;border-radius:4px;overflow-x:auto;"></pre>
                    </div>
                </div>';
    }
private function updateProgress($currentFile) {
        static $lastUpdate = 0;
        $now = microtime(true);
        
        if ($now - $lastUpdate < 0.3) return;
        $lastUpdate = $now;

        echo '<script>
            document.getElementById("progress").innerHTML = `
                <div class="stats">
                    <div class="stat-item">
                        <i class="fas fa-file icon"></i>
                        <strong>Taranan:</strong> ' . number_format($this->count) . '
                    </div>
                    <div class="stat-item">
                        <i class="fas fa-exclamation-triangle icon"></i>
                        <strong>Tehditler:</strong> ' . number_format($this->threats) . '
                    </div>
                    <div class="stat-item">
                        <i class="fas fa-clock icon"></i>
                        <strong>Süre:</strong> ' . $this->getElapsedTime() . '
                    </div>
                    <div class="stat-item">
                        <i class="fas fa-folder-open icon"></i>
                        <strong>Şu an:</strong> ' . htmlspecialchars(basename($currentFile)) . '
                    </div>
                </div>`;
        </script>';
        flush();
        ob_flush();
    }

    public function scan($dir) {
        $this->startTime = microtime(true);
        $this->showHeader();
        echo '<div id="progress" class="progress">Tarama başlatılıyor...</div>';
        flush();
        ob_flush();
        
        $this->scanDir($dir);

        usort($this->foundThreats, function($a, $b) {
            if ($a['count'] != $b['count']) {
                return $b['count'] - $a['count'];
            }
            return strcmp($a['path'], $b['path']);
        });

        foreach ($this->foundThreats as $threat) {
            $this->showThreat($threat);
        }
        
        if ($this->threats === 0) {
            echo '<div class="success" style="background:#4caf50;color:white;padding:20px;border-radius:8px;margin:20px 0;">
                    <i class="fas fa-check-circle icon"></i> Hiç tehdit bulunamadı!
                  </div>';
        }
        
        $this->showFooter();
    }

    private function scanDir($dir) {
        if (!is_readable($dir)) return;
        
        $files = scandir($dir);
        foreach ($files as $file) {
            if ($file == '.' || $file == '..') continue;
            
            $path = $dir . DIRECTORY_SEPARATOR . $file;
            
            if (is_dir($path)) {
                $this->scanDir($path);
            } else {
                $this->checkFile($path, $file);
                $this->updateProgress($path);
            }
        }
    }

    private function checkFile($path, $filename) {
        $this->count++;
        $ext = strtolower(pathinfo($filename, PATHINFO_EXTENSION));
        
        if (!in_array('.' . $ext, $this->suspiciousFiles)) {
            return;
        }

        $matches = [];
        
        foreach ($this->suspiciousNames as $name) {
            if (stripos($filename, $name) !== false) {
                $matches[] = "📁 Şüpheli dosya adı: $name";
            }
        }

        if (is_readable($path)) {
            $content = file_get_contents($path);
            if ($content !== false) {
                foreach ($this->patterns as $pattern => $desc) {
                    if (preg_match("/$pattern/i", $content)) {
                        $matches[] = "⚠️ " . $desc;
                    }
                }
            }
        }

        if (!empty($matches)) {
            $this->foundThreats[] = [
                'path' => $path,
                'matches' => $matches,
                'count' => count($matches),
                'size' => filesize($path),
                'mtime' => filemtime($path),
                'perms' => fileperms($path)
            ];
            $this->threats++;
        }
    }

    private function showThreat($threat) {
        $threatClass = $threat['count'] >= 3 ? 'threat-high' : 
                      ($threat['count'] == 2 ? 'threat-medium' : 'threat-low');
        
        echo '<div class="threat ' . $threatClass . '">
            <h3><i class="fas fa-exclamation-triangle icon"></i> Şüpheli Dosya! 
                <span class="badge">' . $threat['count'] . ' tehdit</span></h3>
            <div class="threat-info">
                <p><i class="fas fa-file icon"></i> <strong>Dosya:</strong> ' . htmlspecialchars($threat['path']) . '</p>
                <p><i class="fas fa-weight icon"></i> <strong>Boyut:</strong> ' . $this->formatSize($threat['size']) . '</p>
                <p><i class="fas fa-clock icon"></i> <strong>Değişiklik:</strong> ' . date("Y-m-d H:i:s", $threat['mtime']) . '</p>
                <p><i class="fas fa-lock icon"></i> <strong>İzinler:</strong> ' . substr(sprintf('%o', $threat['perms']), -4) . '</p>
                <div class="matches">
                    <h4><i class="fas fa-list icon"></i> Tespit Edilen:</h4>';
        foreach ($threat['matches'] as $match) {
            echo '<div class="match-item">• ' . htmlspecialchars($match) . '</div>';
        }
        echo '</div>
            <div class="action-buttons" style="margin-top:10px;">
                <button onclick="deleteFile(\'' . htmlspecialchars($threat['path']) . '\', this)" class="delete-btn">
                    <i class="fas fa-trash icon"></i> Bu Dosyayı Sil
                </button>
                <button onclick="deleteByName(\'' . htmlspecialchars($threat['path']) . '\', \'' . htmlspecialchars(dirname($threat['path'])) . '\')" class="delete-btn" style="background:#c62828;">
                    <i class="fas fa-trash-alt icon"></i> Tüm Aynı İsimli Dosyaları Sil
                </button>
                <button onclick="viewContent(\'' . htmlspecialchars($threat['path']) . '\')" class="view-btn">
                    <i class="fas fa-eye icon"></i> Görüntüle
                </button>
            </div>
        </div>
    </div>';
    }

    private function showFooter() {
        echo '<div class="footer" style="background:#1a237e;color:white;padding:20px;border-radius:8px;margin-top:20px;text-align:center;">
            <h2><i class="fas fa-check-circle icon"></i> Tarama Tamamlandı</h2>
            <div class="stats">
                <div class="stat-item">
                    <i class="fas fa-file icon"></i>
                    <strong>Toplam Taranan:</strong> ' . number_format($this->count) . '
                </div>
                <div class="stat-item">
                    <i class="fas fa-exclamation-triangle icon"></i>
                    <strong>Toplam Tehdit:</strong> ' . number_format($this->threats) . '
                </div>
                <div class="stat-item">
                    <i class="fas fa-clock icon"></i>
                    <strong>Toplam Süre:</strong> ' . $this->getElapsedTime() . '
                </div>
            </div>
            <form method="post" style="margin-top:20px;">
                <button type="submit" class="view-btn">
                    <i class="fas fa-redo icon"></i> Yeni Tarama
                </button>
            </form>
        </div>
        </div></body></html>';
    }

    private function formatSize($size) {
        $units = ['B', 'KB', 'MB', 'GB'];
        $power = floor(($size ? log($size) : 0) / log(1024));
        return sprintf("%.2f %s", $size / pow(1024, $power), $units[$power]);
    }

    private function getElapsedTime() {
        $elapsed = microtime(true) - $this->startTime;
        $hours = floor($elapsed / 3600);
        $minutes = floor(($elapsed % 3600) / 60);
        $seconds = floor($elapsed % 60);
        return sprintf("%02d:%02d:%02d", $hours, $minutes, $seconds);
    }

    public function showStartForm() {
        $this->showHeader();
        echo '<div class="input-form" style="background:#fff;padding:20px;border-radius:8px;box-shadow:0 2px 4px rgba(0,0,0,0.1);">
                <h2><i class="fas fa-folder-open icon"></i> Tarama Başlat</h2>
                <form method="post" style="margin-top:20px;">
                    <div style="margin-bottom:15px;">
                        <label for="root_dir" style="display:block;margin-bottom:5px;">
                            <i class="fas fa-sitemap icon"></i> Web Root Dizini:
                        </label>
                        <input type="text" name="root_dir" id="root_dir" 
                               value="' . htmlspecialchars(dirname(getcwd())) . '" required
                               style="width:100%;padding:8px;border:1px solid #ddd;border-radius:4px;margin-bottom:15px;">
                        
                        <label for="scan_dir" style="display:block;margin-bottom:5px;">
                            <i class="fas fa-folder icon"></i> Taranacak Dizin:
                        </label>
                        <input type="text" name="scan_dir" id="scan_dir" 
                               value="' . htmlspecialchars(getcwd()) . '" required
                               style="width:100%;padding:8px;border:1px solid #ddd;border-radius:4px;">
                    </div>
                    <button type="submit" class="view-btn" style="width:100%;">
                        <i class="fas fa-play icon"></i> Taramayı Başlat
                    </button>
                </form>
            </div>
            </div></body></html>';
    }
} 


if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    if (isset($_POST['scan_dir'])) {
        $scanDir = realpath($_POST['scan_dir']);
        if ($scanDir === false || !is_dir($scanDir)) {
            die('Geçersiz dizin yolu!');
        }
        $scanner = new ShellScanner();
        $scanner->scan($scanDir);
        exit;
    } elseif (isset($_POST['view_content'])) {
        $filePath = $_POST['view_content'];
        if (file_exists($filePath) && is_file($filePath) && is_readable($filePath)) {
            $content = file_get_contents($filePath);
            if ($content !== false) {
                echo json_encode([
                    'success' => true,
                    'content' => htmlspecialchars($content)
                ]);
            } else {
                echo json_encode([
                    'success' => false,
                    'message' => 'Dosya içeriği okunamadı'
                ]);
            }
        } else {
            echo json_encode([
                'success' => false,
                'message' => 'Dosya bulunamadı veya okunamıyor'
            ]);
        }
        exit;
    } elseif (isset($_POST['delete_file'])) {
        $filePath = $_POST['delete_file'];
        if (file_exists($filePath) && is_file($filePath)) {
            if (unlink($filePath)) {
                echo json_encode(['success' => true, 'message' => 'Dosya başarıyla silindi']);
            } else {
                echo json_encode(['success' => false, 'message' => 'Dosya silinemedi']);
            }
        }
        exit;
    } elseif (isset($_POST['delete_by_name'])) {
        $fileName = basename($_POST['delete_by_name']);
        $rootPath = $_POST['root_dir'];
        
        if (!is_dir($rootPath)) {
            echo json_encode([
                'success' => false,
                'message' => 'Geçersiz root dizin!'
            ]);
            exit;
        }
        
        $deletedFiles = [];
        $errors = [];
        
        try {
            $iterator = new RecursiveIteratorIterator(
                new RecursiveDirectoryIterator($rootPath),
                RecursiveIteratorIterator::SELF_FIRST,
                RecursiveIteratorIterator::CATCH_GET_CHILD
            );
    
            foreach ($iterator as $file) {
                if ($file->getFilename() === '.' || 
                    $file->getFilename() === '..' ||
                    strpos($file->getPathname(), '/.') !== false) {
                    continue;
                }
    
                if ($file->isFile() && $file->getFilename() === $fileName) {
                    $path = $file->getRealPath();
                    try {
                        if (unlink($path)) {
                            $deletedFiles[] = $path;
                        } else {
                            $errors[] = $path;
                        }
                    } catch (Exception $e) {
                        $errors[] = $path . " (" . $e->getMessage() . ")";
                    }
                }
            }
        } catch (Exception $e) {
            $errors[] = "Tarama hatası: " . $e->getMessage();
        }
        
        echo json_encode([
            'success' => true,
            'deletedFiles' => $deletedFiles,
            'errors' => $errors,
            'message' => sprintf('Toplam %d dosya silindi, %d hata oluştu', 
                count($deletedFiles), count($errors))
        ]);
        exit;
    }
}


if ($_SERVER['REQUEST_METHOD'] !== 'POST' || !isset($_POST['scan_dir'])) {
    $scanner = new ShellScanner();
    $scanner->showStartForm();
}
?>
