https://github.com/Sorvigolova/ipatool

# download
git clone https://github.com/Sorvigolova/ipatool.git
cd ipatool

# install
brew install openssl@3 minizip nlohmann-json unicorn pkg-config
cmake -B build -DCMAKE_BUILD_TYPE=Release \                         
      -DOPENSSL_ROOT_DIR=$(brew --prefix openssl@3) \
      -DCMAKE_OSX_ARCHITECTURES=arm64
cmake --build build

# get ipa
build/ipatool auth login
build/ipatool search <AppName>
build/ipatool download -i <AppId> -o <OutputDir>

# find app in ipa
mkdir unpack
ditto -x -k <FileName>.ipa unpack
find unpack/Payload -name "*.app"

# install ghidra for mac
git clone https://github.com/ytisf/GhidraMacOS
cd GhidraMacOS
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python3 install_ghidra.py
open ghidra_install/Ghidra.app
