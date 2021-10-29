---
title: oh_my_zsh_theme
date: 2020-10-09 16:20:18
tags:
---
```bash
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
sudo easy_install pip
pip install powerline-status

# clone
git clone https://github.com/powerline/fonts.git --depth=1
# install
cd fonts
./install.sh
# clean-up a bit
cd ..
rm -rf fonts
#edit iterm->preference->text->font
#check use difference font for non-ascii text and change both font

vim ~/.zshrc
ZSH_THEME="agnoster"

brew install zsh-syntax-highlighting

echo 'source /usr/local/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh' >> ~/.zshrc

# or use 
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

~/.zshrc add plugin
plugins=(
  git
  zsh-autosuggestions
  zsh-syntax-highlighting
)
```