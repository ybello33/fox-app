# Use a lightweight Python image
FROM python:3.9-slim

# Set the working directory in the container
WORKDIR /app

# Copy the requirements file
COPY requirements.txt /app/

# Install required Python libraries
RUN pip install --no-cache-dir -r requirements.txt

# Copy the application source code
COPY main.py /app/

# Expose the port the application will run on
EXPOSE 8000

# Command to run the application
CMD ["python", "main.py"]
