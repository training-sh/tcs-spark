One time env addition to .bashrc

This .bashrc is loaded every time you login/open new terminal

```
tee -a ~/.bashrc > /dev/null <<'EOF'
export AWS_ACCESS_KEY_ID=team
export AWS_SECRET_ACCESS_KEY=team1234
export AWS_DEFAULT_REGION=us-east-1
export POLARIS_URL="http://localhost:8181"

echo "AWS & Polaris env set"
EOF
```

if first time on same terminal, not needed for new terminal/after login

```
source ~/.bashrc
```


