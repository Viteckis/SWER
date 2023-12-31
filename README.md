# SWER
radio
name: Linux

on: [push, pull_request]

jobs:
  test_linux:
    runs-on: "ubuntu-20.04"
    name: "Ubuntu latest - Python ${{ matrix.python-version }}"
    env:
      USING_COVERAGE: '3.11'

    strategy:
      matrix:
        python-version: ["2.7", "3.5", "3.6", "3.7", "3.8", "3.9", "3.10", "3.11", "pypy2.7", "pypy3.7", "pypy3.8", "pypy3.9"]
        redis-version: [6]

    steps:
      - uses: "actions/checkout@v3"
      - uses: "actions/setup-python@v4"
        with:
          python-version: "${{ matrix.python-version }}"
          cache: "pip"
          cache-dependency-path: |
            **/setup.py
            **/requirements*.txt

      - name: "Install dependencies"
        run: |
          set -xe
          sudo apt-get install -y libxml2-dev libxslt-dev
          python -VV
          python -m pip install --upgrade pip setuptools wheel codecov
          python -m pip install -e .
          python -m pip install -r requirements-pytest.txt
          python -m pip install -r requirements-docs.txt

      - name: Start Redis
        uses: supercharge/redis-github-action@1.2.0
        with:
          redis-version: ${{ matrix.redis-version }}

      - name: "Run tests for ${{ matrix.python-version }}"
        env: 
          CLUBLOG_APIKEY: ${{ secrets.CLUBLOG_APIKEY }}
          QRZ_USERNAME: ${{ secrets.QRZ_USERNAME }}
          QRZ_PWD: ${{ secrets.QRZ_PWD }}
          PYTHON_VERSION: ${{ matrix.python-version }}
        # delay the execution randomly by a couple of seconds to reduce the amount 
        # of concurrent API calls on Clublog and QRZ.com when all CI jobs execute simultaneously
        run: |
          sleep $[ ( $RANDOM % 10 )  + 1 ]s 
          pytest --cov=./
          if [[ $PYTHON_VERSION == 3.11 ]]; then codecov; fi
          cd docs && make html

  # publish_package:
  #   runs-on: "ubuntu-latest"
  #   needs: ["test_linux"]
  #   if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags')
  #   steps: 
  #     - name: Publish package
  #       uses: pypa/gh-action-pypi-publish@release/v1
  #       with:
  #         user: __token__
  #         password: ${{ secrets.PYPI_API_TOKEN }}

  test_windows:
    runs-on: "windows-latest"
    name: "Windows latest - Python ${{ matrix.python-version }}"

    strategy:
      matrix:
        # lxml support for windows/python3.11 still missing (Dec 2022)
        # https://github.com/lxml/lxml/pull/360
        # python-version: ["3.6", "3.7", "3.8", "3.9", "3.10", "3.11"]
        # python-version: ["3.6", "3.7", "3.8", "3.9", "3.10"]
        # disable python 3.6 on windows due to lxml / bs4 problems on Github Actions
        python-version: ["3.7", "3.8", "3.9", "3.10"]

    steps:
      - uses: "actions/checkout@v3"
      - uses: "actions/setup-python@v4"
        with:
          python-version: "${{ matrix.python-version }}"
          cache: "pip"
          cache-dependency-path: |
                setup.py
                requirements*.txt
      - name: "Install dependencies"
        run: |
          python -VV
          python -m pip install --upgrade pip setuptools wheel codecov
          python -m pip install -e .
          python -m pip install -r requirements-pytest.txt
          python -m pip install -r requirements-docs.txt
      - name: Setup redis
        # We have to download and install a non-official redis windows port
        # since there is no official redis version for windows.
        # 5.0 is good enough for our purposes. After installing the msi, 
        # redis will startup as a service. 
        # There are no github-actions supporting redis on windows.
        # Github Actions Container services are also not available for windows.
        run: |
          C:\msys64\usr\bin\wget.exe https://github.com/tporadowski/redis/releases/download/v5.0.14.1/Redis-x64-5.0.14.1.msi
          msiexec /quiet /i Redis-x64-5.0.14.1.msi
      - name: "Run tests for ${{ matrix.python-version }}"
        env: 
          CLUBLOG_APIKEY: ${{ secrets.CLUBLOG_APIKEY }}
          QRZ_USERNAME: ${{ secrets.QRZ_USERNAME }}
          QRZ_PWD: ${{ secrets.QRZ_PWD }}
          PYTHON_VERSION: ${{ matrix.python-version }}
        # delay the execution randomly by 1-20sec to reduce the
        # amount of concurrent API calls on Clublog and QRZ.com
        # when all CI jobs execute simultaneously
        run: |
          start-sleep -Seconds (1..10 | get-random)
          pytest
