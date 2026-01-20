#Fargateコンテナのため、SSM Agentが入っているイメージを使用
FROM public.ecr.aws/amazonlinux/amazonlinux:2023.8.20250721.2

#必要なパッケージのインストール
RUN yum update && yum install -y  tar unzip gzip openssl less vi git lsof procps iproute jq 

# kubectl(Kubernetes1.32)のインストール
RUN curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.32.0/2024-12-20/bin/linux/amd64/kubectl \
  && openssl sha1 -sha256 kubectl \
  && chmod +x ./kubectl \
  && mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH \
  && echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
# Helm CLIのインストール
RUN curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \
  && chmod 700 get_helm.sh \
  && ./get_helm.sh
# Postgresqlクライアントのインストール
RUN yum install -y postgresql15.x86_64
# AWS CLIのインストール
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
  && unzip awscliv2.zip \
  && ./aws/install
# Argo CD CLIのインストール
RUN curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 \
  && install -m 555 argocd-linux-amd64 /usr/local/bin/argocd \
  && rm argocd-linux-amd64
# Aliasの設定
RUN echo 'alias ll="ls -l"' >> ~/.bashrc \
  && echo 'alias k="kubectl"' >> ~/.bashrc
# 非rootユーザーを作成
RUN useradd -m -s /bin/bash user
# 非rootユーザーに切り替え
USER user
# 常時起動のため終了しない&何も起動しないプロセスを起動
ENTRYPOINT [ "tail", "-F", "/dev/null" ]