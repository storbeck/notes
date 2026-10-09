https://github.com/Sorvigolova/ipatool

# download
```bash
git clone https://github.com/Sorvigolova/ipatool.git
cd ipatool
```

# install
```bash
brew install openssl@3 minizip nlohmann-json unicorn pkg-config
cmake -B build -DCMAKE_BUILD_TYPE=Release \                         
      -DOPENSSL_ROOT_DIR=$(brew --prefix openssl@3) \
      -DCMAKE_OSX_ARCHITECTURES=arm64
cmake --build build
```

# get ipa
```bash
build/ipatool auth login
build/ipatool search <AppName>
build/ipatool download -i <AppId> -o <OutputDir>
```

# find app in ipa
```bash
mkdir unpack
ditto -x -k <FileName>.ipa unpack
find unpack/Payload -name "*.app"
```

# install ghidra for mac
```bash
git clone https://github.com/ytisf/GhidraMacOS
cd GhidraMacOS
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python3 install_ghidra.py
open ghidra_install/Ghidra.app
```

# check if encrypted
If `crypted 1` then it's FairPlay-encrypted.

```bash
otool -l <Binary> | grep -A5 -E 'LC_ENCRYPTION_INFO(_64)?'
          cmd LC_ENCRYPTION_INFO_64
      cmdsize 24
     cryptoff 16384
    cryptsize 180224
      cryptid 1
          pad 0
```
