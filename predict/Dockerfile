#ARG model_chip 
FROM fluxcapacitor/package-tensorflow-7a7fe93-4c0052d-cpu:master
# This needs to come after the FROM
#RUN \
#  echo $model_chip
ENV \
  PIPELINE_MODEL_CHIP=cpu

RUN rm /bin/sh && ln -s /bin/bash /bin/sh

WORKDIR /root

RUN \
  apt-get update

# Create an app user so our program doesn't run as root.
RUN groupadd -r appuser \
  && useradd -r -g appuser -d /root -p password -c "app user" appuser \
  && usermod -aG sudo appuser

RUN \
  apt-get install -y sudo

RUN \
  echo "appuser ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers \
  && cat /etc/sudoers

RUN \
  chmod a+wrx /root

RUN \
  echo "deb https://dl.bintray.com/sbt/debian /" | tee -a /etc/apt/sources.list.d/sbt.list \
  && apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2EE0EA64E40A89B84B2DF73499E82A75642AC823 \
  && apt-get update \
  && apt-get install -y sbt

# Anaconda's libgfortran=3.0 is not co-operating, so we use apt-get
RUN \
  apt-get install libgfortran3
RUN \
  conda install -c anaconda openblas

# We're installing (duplicate of base Docker image)
#   in order to force an error during the Docker build
#   if/when the base Docker image changes TF version.
# When the base Docker image changes TF version, be sure
#   to propagate this change to the following:
#     src/main/python/model_server/requirements/python/requirements_pipeline_model_server.txt
RUN \
  pip install /tmp/pip/tensorflow-1.2.0rc2-cp36-cp36m-linux_x86_64.whl \
  && pip uninstall -y /tmp/pip/tensorflow-1.2.0rc2-cp36-cp36m-linux_x86_64.whl

RUN \
  apt-get install -y nginx 

RUN \
  service nginx start

RUN \
  apt-get install -y tmux \
  && apt-get install -y screen

RUN \
  wget https://github.com/prometheus/prometheus/releases/download/v1.7.1/prometheus-1.7.1.linux-amd64.tar.gz \
  && tar xvfz prometheus-1.7.1.linux-amd64.tar.gz \
  && rm prometheus-1.7.1.linux-amd64.tar.gz

RUN \
  wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana_4.3.1_amd64.deb \
  && apt-get install -y adduser libfontconfig \
  && dpkg -i grafana_4.3.1_amd64.deb \
  && rm grafana_4.3.1_amd64.deb

RUN \
  service grafana-server start

#RUN \
#  mv /etc/grafana/grafana.ini /etc/grafana/grafana.ini.orig \
#  && cd /etc/grafana/ \
#  && ln -s /root/config/grafana/grafana.ini

RUN \
  mkdir -p /root/logs

ENV \
  LOGS_HOME=/root/logs

COPY sysutils/ sysutils/

ENV \
  PIPELINE_JVM_MODEL_SERVER_PATH=/root/jvm
COPY jvm/ jvm/
RUN \
  cd $PIPELINE_JVM_MODEL_SERVER_PATH && sbt clean-files package
RUN \
  cd $PIPELINE_JVM_MODEL_SERVER_PATH/lib/jni \
  && ln -s libtensorflow_jni-$PIPELINE_MODEL_CHIP.so libtensorflow_jni.so

COPY config/ config/

RUN \
  mv /root/prometheus-1.7.1.linux-amd64/prometheus.yml /root/prometheus-1.7.1.linux-amd64/prometheus.yml.orig \
  && cd /root/prometheus-1.7.1.linux-amd64/ \
  && ln -s /root/config/prometheus/prometheus.yml

ENV \
  PATH=/root/prometheus-1.7.1.linux-amd64/:$PATH
RUN \
  mv /etc/nginx/sites-available/default /etc/nginx/sites-available/default.orig \
  && cd /etc/nginx/sites-available/ \
  && ln -s /root/config/nginx/default \
  && cd /etc/nginx/sites-enabled/ \
  && rm default \
  && ln -s /etc/nginx/sites-available/default

COPY src/ src/

ENV \
  PIPELINE_MODEL_SERVER_PATH=/root/src/main/python/model_server

RUN \
  apt-get install -y erlang erlang-ssl erlang-inets erlang-dev \
    erlang-edoc erlang-eunit erlang-tools erlang-common-test

RUN \
  apt-get install --yes sqlite3 libsqlite3-dev

RUN \
  apt-get install --yes nodejs-legacy npm

RUN \
  echo '{ "allow_root": true }' > /root/.bowerrc \
  && npm install -g bower

COPY guild/ guild/
RUN \
  cd guild \
  && make
RUN \
  ln -s /root/guild/scripts/guild-dev /usr/local/bin/guild

ENV \
  PIPELINE_MODEL_SERVER_PORT=9876

ENV \
  PIPELINE_MODEL_SERVER_PROMETHEUS_PORT=10254

ENV \
  PIPELINE_MODEL_SERVER_TENSORFLOW_PORT=9877

ENV \
  PIPELINE_MODEL_SERVER_TENSORFLOW_PROMETHEUS_PORT=10255

ENV \
  PIPELINE_MODEL_SERVER_TENSORFLOW_SERVING_PORT=9000

ENV \
  PIPELINE_MODEL_SERVER_TENSORFLOW_SERVING_REQUEST_BATCHING=true

ENV \
  TF_CPP_MIN_LOG_LEVEL=0

COPY html/ /var/www/html/
COPY run run

ARG model_type
RUN \
  echo $model_type
ENV \
  PIPELINE_MODEL_TYPE=$model_type

ARG model_name
RUN \
  echo $model_name
ENV \
  PIPELINE_MODEL_NAME=$model_name

ARG model_tag
RUN \
  echo $model_tag

ARG model_path
RUN \
  echo $model_path
ENV \
  PIPELINE_MODEL_PATH=/root/model

RUN \
  pip install awscli

COPY $model_path $PIPELINE_MODEL_PATH
#RUN \
#  if [[ $model_path == "http:"* ]] || [[ $model_path == "https:"* ]]; then \
#    wget -r -e --no-parent -O $PIPELINE_MODEL_PATH $model_path; \
#  fi 

#RUN \
#  if [[ $model_path == "s3:"* ]]; then \
#    aws s3 cp --no-sign-request --recursive \
#      $model_path \
#      $PIPELINE_MODEL_PATH; \
#  fi 

ENV \
  PIPELINE_CONDA_ENV_NAME=pipeline

RUN \
  if [[ $PIPELINE_MODEL_TYPE == "python" ]] || \
     [[ $PIPELINE_MODEL_TYPE == "keras" ]] || \
     [[ $PIPELINE_MODEL_TYPE == "scikit" ]] || \
     [[ $PIPELINE_MODEL_TYPE == "tensorflow" ]]; then \
       echo "" \
       && echo "Creating and Activating '$PIPELINE_CONDA_ENV_NAME' Conda Environment with '$PIPELINE_MODEL_TYPE/$PIPELINE_MODEL_NAME' Model Dependencies with '$PIPELINE_MODEL_PATH/pipeline_conda_environment.yml'..." \
       && echo "" \
       && conda env create --name $PIPELINE_CONDA_ENV_NAME \
         --file $PIPELINE_MODEL_PATH/pipeline_conda_environment.yml \
       && echo "" \
       && echo "...Created and Activated!" \
       && echo "" \
       && echo "" \
       && echo "Installing Model Server Dependencies..." \
       && echo "" \
       && conda env update --name $PIPELINE_CONDA_ENV_NAME \ 
         --file $PIPELINE_MODEL_SERVER_PATH/requirements/pipeline_model_server_conda_environment.yml \
       && echo "" \
       && echo "...Model Server Dependencies Installed!" \
       && echo ""; \
  fi

# Guild/TensorFlow Requirements
RUN \
  if [[ $PIPELINE_MODEL_TYPE == "tensorflow" ]]; then \
      source activate $PIPELINE_CONDA_ENV_NAME \
      && pip install psutil \
      && source activate $PIPELINE_CONDA_ENV_NAME \
      && pip2 install tensorflow==1.2.0 \
      && conda list -n $PIPELINE_CONDA_ENV_NAME; \
  fi

ENV SHELL=/bin/bash
RUN \
  chmod a+wr -R /opt/conda/lib/python3.6/site-packages \
  && chmod a+wr -R /root/.cache/pip

RUN \
  chmod a+wr -R /root/model \
  && chmod a+wr -R /root/.ivy2 \
  && chmod a+wr -R /root/.sbt

USER appuser

EXPOSE 6969 7070 9876 9877 9000 9001 10254 10255 9040 9090 3000 6333

CMD ["supervise", "."]
