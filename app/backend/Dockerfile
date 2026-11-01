#use image of pyton
FROM python:3.11-slim
#cd for order all the files would be in this path
WORKDIR /app
#copy the file with liberies to this path (/app/)
COPY requirements.txt .
# install the libraries (read from requirements.txt)
# pip skip ssl search in gogle for skiping checking of netfree
RUN pip install --trusted-host pypi.org --trusted-host files.pythonhosted.org  --no-cache-dir -r requirements.txt
#put my code for run this last change frquency
COPY app.py .
# the port un the dockerfile
EXPOSE 5000
#after build this will be runing and show the application directly!
CMD [ "python","app.py" ]