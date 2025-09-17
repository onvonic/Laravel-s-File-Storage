# Laravel File Upload Documentation

## Overview
Dokumentasi ini menjelaskan cara mengimplementasikan upload file di Laravel dengan berbagai opsi penyimpanan dan validasi.

## Setup Controller

```php
<?php

namespace App\Http\Controllers\Dash\Module\Master;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Validator;
use Illuminate\Support\Facades\Storage;

class MasterCompanyController extends Controller
{
    // Controller methods here
}
```

## File Validation

### Basic File Validation
```php
$validator = Validator::make($request->all(), [
    'file_logo' => 'nullable|file|mimetypes:image/svg+xml,image/jpeg,image/png,image/jpg|max:500',
]);

if ($validator->fails()) {
    return response()->json([
        'status'  => false,
        'type'    => 'info',
        'message' => 'Validation error.',
        'error'   => $validator->errors()
    ]);
}
```

### Validation Rules Explanation
- `nullable` - File bersifat opsional
- `file` - Input harus berupa file
- `mimetypes` - Tipe file yang diizinkan (SVG, JPEG, PNG, JPG)
- `max:500` - Ukuran maksimal 500KB

### Image Ratio Validation
```php
$files_input = $request->file('file_logo');

// Cek rasio gambar
$files_inputSize = getimagesize($files_input);
if ($files_inputSize) {
    $width       = $files_inputSize[0];
    $height      = $files_inputSize[1];
    $targetRatio = 16 / 9;
    $actualRatio = $width / $height;
    $tolerance   = 0.01; // 1% tolerance
    
    if (abs($actualRatio - $targetRatio) > $tolerance) {
        return response()->json([
            'status'  => false,
            'message' => 'Validation error.',
            'error'   => ['file_logo' => ['Image must have a 16:9 ratio']]
        ]);
    }
}
```

## File Storage Options

### 1. Public Storage (Accessible via URL)
```php
$save_mode = 'public';

if ($save_mode === 'public') {
    // Buat directory jika belum ada
    if (!Storage::disk('public')->exists('files/company')) {
        Storage::disk('public')->makeDirectory('files/company');
    }
    
    // Simpan file
    $files_input_path = $files_input->storeAs('files/company', $files_input_name, 'public');
    $files_input_url  = asset('storage/' . $files_input_path);
}
```

**Karakteristik:**
- ✅ Cepat dan langsung bisa diakses via URL
- ✅ Cocok untuk gambar/file publik
- ❌ Kurang aman untuk file sensitif
- Path: `storage/app/public/files/company/`
- URL: `http://domain.com/storage/files/company/filename.jpg`

### 2. Private Storage (Controlled Access)
```php
$save_mode = 'local';

if ($save_mode === 'local') {
    // Buat directory jika belum ada
    if (!Storage::disk('local')->exists('files/company')) {
        Storage::disk('local')->makeDirectory('files/company');
    }
    
    // Simpan file
    $files_input_path = $files_input->storeAs('files/company', $files_input_name, 'local');
    $files_input_url  = $files_input_path;
}
```

**Karakteristik:**
- ✅ Lebih aman dengan kontrol akses
- ✅ File tidak bisa diakses langsung via URL
- ❌ Perlu route khusus untuk mengakses file
- Path: `storage/app/files/company/`

### 3. Default Image Handler
```php
if (!$request->hasFile('file_logo')) {
    $files_input_url = asset('assets/image/no_image_available.jpg');
}
```

## File Naming Convention

```php
if ($request->hasFile('file_logo')) {
    $files_input = $request->file('file_logo');
    $files_input_name = 'post' . now()->format('YmdHis') . '_' . bin2hex(random_bytes(8)) . '.' . $files_input->getClientOriginalExtension();
}
```

**Format:** `post20241217143025_a1b2c3d4e5f6g7h8.jpg`
- `post` - Prefix
- `20241217143025` - Timestamp (YmdHis)
- `a1b2c3d4e5f6g7h8` - Random hex (8 bytes)
- `.jpg` - Original extension

## Database Storage

```php
DB::table('mst_files')->insert([
    'file_logo'        => $files_input_url, // File path/URL
    'created_by'       => Auth::id(),
]);
```

## Complete Implementation Example

```php
public function insert(Request $request)
{
    // Validasi input
    $validator = Validator::make($request->all(), [
        'file_logo' => 'nullable|file|mimetypes:image/svg+xml,image/jpeg,image/png,image/jpg|max:500',
    ]);
    
    if ($validator->fails()) {
        return response()->json([
            'status'  => false,
            'type'    => 'info',
            'message' => 'Validation error.',
            'error'   => $validator->errors()
        ]);
    }

    DB::beginTransaction();
    try {
        $files_input_url = asset('assets/image/no_image_available.jpg'); // Default
        
        if ($request->hasFile('file_logo')) {
            $files_input = $request->file('file_logo');
            
            // Generate unique filename
            $files_input_name = 'post' . now()->format('YmdHis') . '_' . bin2hex(random_bytes(8)) . '.' . $files_input->getClientOriginalExtension();
            
            // Choose storage mode
            $save_mode = 'local'; // or 'public'
            
            if ($save_mode === 'public') {
                // Public storage
                if (!Storage::disk('public')->exists('files/company')) {
                    Storage::disk('public')->makeDirectory('files/company');
                }
                $files_input_path = $files_input->storeAs('files/company', $files_input_name, 'public');
                $files_input_url = asset('storage/' . $files_input_path);
            } else {
                // Private storage
                if (!Storage::disk('local')->exists('files/company')) {
                    Storage::disk('local')->makeDirectory('files/company');
                }
                $files_input_path = $files_input->storeAs('files/company', $files_input_name, 'local');
                $files_input_url = $files_input_path;
            }
        }

        // Simpan ke database
        DB::table('mst_file')->insert([
            'file_logo'  => $files_input_url,
            'created_by' => Auth::id(),
        ]);

        DB::commit();

        return response()->json([
            'status'  => true,
            'type'    => 'success',
            'message' => 'File uploaded successfully',
        ]);
        
    } catch (\Exception $e) {
        DB::rollback();
        return response()->json([
            'status'  => false,
            'type'    => 'error',
            'message' => 'Upload failed: ' . $e->getMessage(),
        ]);
    }
}
```

## Additional Validation Rules

```php
// Ukuran file maksimal
'file' => 'max:2048', // 2MB

// Dimensi gambar
'image' => 'dimensions:min_width=100,min_height=100,max_width=2000,max_height=2000',

// Multiple file types
'document' => 'mimes:pdf,doc,docx,txt',

// Required file
'avatar' => 'required|image|max:1024',
```

## Storage Configuration

Pastikan konfigurasi storage di `config/filesystems.php`:

```php
'disks' => [
    'local' => [
        'driver' => 'local',
        'root' => storage_path('app'),
    ],

    'public' => [
        'driver' => 'local',
        'root' => storage_path('app/public'),
        'url' => env('APP_URL').'/storage',
        'visibility' => 'public',
    ],
],
```

## Best Practices

1. **Selalu gunakan validasi** untuk keamanan
2. **Generate unique filename** untuk menghindari konflik
3. **Gunakan database transaction** untuk konsistensi data
4. **Handle exceptions** dengan proper error response
5. **Pilih storage yang tepat** sesuai kebutuhan (public vs private)
6. **Batasi ukuran dan tipe file** sesuai kebutuhan aplikasi
7. **Gunakan middleware** untuk autentikasi dan autorisasi
