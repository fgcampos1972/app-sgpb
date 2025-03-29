import os
import logging

from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy.orm import DeclarativeBase
from flask_login import LoginManager
from flask_bcrypt import Bcrypt

# Configure logging
logging.basicConfig(level=logging.DEBUG)

class Base(DeclarativeBase):
    pass

db = SQLAlchemy(model_class=Base)

# create the app
app = Flask(__name__)
app.secret_key = os.environ.get("SESSION_SECRET", "dev_secret_key")

# configure the database, relative to the app instance folder
app.config["SQLALCHEMY_DATABASE_URI"] = os.environ.get("DATABASE_URL", "sqlite:///bens_patrimoniais.db")
app.config["SQLALCHEMY_ENGINE_OPTIONS"] = {
    "pool_recycle": 300,
    "pool_pre_ping": True,
}
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

# Initialize the app with the extension
db.init_app(app)

# Initialize Bcrypt
bcrypt = Bcrypt(app)

# Initialize Login Manager
login_manager = LoginManager(app)
login_manager.login_view = 'login'
login_manager.login_message = 'Por favor, faça login para acessar esta página.'
login_manager.login_message_category = 'info'

@login_manager.user_loader
def load_user(user_id):
    from models import User
    return User.query.get(int(user_id))

with app.app_context():
    # Make sure to import the models here or their tables won't be created
    import models
    db.create_all()
    
# Função para criar o usuário administrador
def create_admin_user(username, email, password):
    with app.app_context():
        from models import User
        admin = User.query.filter_by(username=username).first()
        if not admin:
            admin = User(username=username, email=email, role='admin', is_active=True)
            admin.set_password(password)
            db.session.add(admin)
            db.session.commit()
            app.logger.info(f"Usuário administrador '{username}' criado com sucesso.")
            return True
        else:
            app.logger.info(f"Usuário '{username}' já existe.")
            return False

from app import db
from datetime import datetime
from flask_login import UserMixin

class BemPatrimonial(db.Model):
    __tablename__ = 'bens_patrimoniais'
    
    id = db.Column(db.Integer, primary_key=True)
    regional = db.Column(db.String(100), nullable=False)
    municipio = db.Column(db.String(100), nullable=False)
    comunidade = db.Column(db.String(100), nullable=False)
    responsavel = db.Column(db.String(100), nullable=False)
    termo = db.Column(db.String(100), nullable=True)
    vigencia = db.Column(db.String(100), nullable=True)
    item = db.Column(db.String(100), nullable=False)
    especificacao = db.Column(db.Text, nullable=True)
    patrimonio = db.Column(db.String(100), nullable=True)
    marca = db.Column(db.String(100), nullable=True)
    capacidade = db.Column(db.String(100), nullable=True)
    placa_serie = db.Column(db.String(100), nullable=True)
    pode_ser_doado = db.Column(db.String(3), nullable=True)  # Sim ou Não
    situacao_bem = db.Column(db.String(100), nullable=True)
    processo = db.Column(db.String(100), nullable=True)
    situacao_processo = db.Column(db.String(100), nullable=True)
    data_fiscalizacao = db.Column(db.Date, nullable=True)
    fiscalizador = db.Column(db.String(100), nullable=True)
    data_cadastro = db.Column(db.DateTime, default=datetime.utcnow)
    
    def __repr__(self):
        return f'<BemPatrimonial {self.id} - {self.item}>'
    
    def to_dict(self):
        return {
            'id': self.id,
            'regional': self.regional,
            'municipio': self.municipio,
            'comunidade': self.comunidade,
            'responsavel': self.responsavel,
            'termo': self.termo,
            'vigencia': self.vigencia,
            'item': self.item,
            'especificacao': self.especificacao,
            'patrimonio': self.patrimonio,
            'marca': self.marca,
            'capacidade': self.capacidade,
            'placa_serie': self.placa_serie,
            'pode_ser_doado': self.pode_ser_doado,
            'situacao_bem': self.situacao_bem,
            'processo': self.processo,
            'situacao_processo': self.situacao_processo,
            'data_fiscalizacao': self.data_fiscalizacao.strftime('%d/%m/%Y') if self.data_fiscalizacao else '',
            'fiscalizador': self.fiscalizador,
            'data_cadastro': self.data_cadastro.strftime('%d/%m/%Y %H:%M')
        }

class User(UserMixin, db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(256), nullable=False)
    role = db.Column(db.String(20), default='operador')  # 'admin', 'gerente', 'operador', 'leitor'
    is_active = db.Column(db.Boolean, default=True)
    last_login = db.Column(db.DateTime, nullable=True)
    date_created = db.Column(db.DateTime, default=datetime.utcnow)
    
    def __repr__(self):
        return f'<User {self.username}>'
    
    def set_password(self, password):
        from app import bcrypt
        self.password_hash = bcrypt.generate_password_hash(password).decode('utf-8')
        
    def check_password(self, password):
        from app import bcrypt
        return bcrypt.check_password_hash(self.password_hash, password)
    
    def is_admin(self):
        return self.role == 'admin'
    
    def is_gerente(self):
        return self.role == 'gerente' or self.role == 'admin'

import os
import tempfile
import random
import pandas as pd
from flask import render_template, request, redirect, url_for, flash, jsonify, send_file
from datetime import datetime
from sqlalchemy import or_, func
from werkzeug.utils import secure_filename
from flask_login import login_user, logout_user, login_required, current_user

from app import app, db, create_admin_user
from models import BemPatrimonial, User
from utils import export_to_xlsx, create_excel_modelo

# Adicionar a variável 'now' para todos os templates
@app.context_processor
def inject_now():
    return {'now': datetime.now()}

@app.route('/')
@login_required
def index():
    return render_template('index.html')

@app.route('/api/bens', methods=['GET'])
@login_required
def get_bens():
    draw = request.args.get('draw', type=int)
    start = request.args.get('start', type=int, default=0)
    length = request.args.get('length', type=int, default=10)
    search = request.args.get('search[value]', type=str, default='')
    
    # Base query
    query = BemPatrimonial.query
    
    # Apply search if provided
    if search:
        query = query.filter(
            or_(
                BemPatrimonial.regional.ilike(f'%{search}%'),
                BemPatrimonial.municipio.ilike(f'%{search}%'),
                BemPatrimonial.comunidade.ilike(f'%{search}%'),
                BemPatrimonial.responsavel.ilike(f'%{search}%'),
                BemPatrimonial.termo.ilike(f'%{search}%'),
                BemPatrimonial.vigencia.ilike(f'%{search}%'),
                BemPatrimonial.item.ilike(f'%{search}%'),
                BemPatrimonial.especificacao.ilike(f'%{search}%'),
                BemPatrimonial.patrimonio.ilike(f'%{search}%'),
                BemPatrimonial.marca.ilike(f'%{search}%'),
                BemPatrimonial.capacidade.ilike(f'%{search}%'),
                BemPatrimonial.placa_serie.ilike(f'%{search}%'),
                BemPatrimonial.pode_ser_doado.ilike(f'%{search}%'),
                BemPatrimonial.situacao_bem.ilike(f'%{search}%'),
                BemPatrimonial.processo.ilike(f'%{search}%'),
                BemPatrimonial.situacao_processo.ilike(f'%{search}%'),
                BemPatrimonial.fiscalizador.ilike(f'%{search}%')
            )
        )
    
    # Apply additional filters if provided
    regional = request.args.get('regional')
    if regional:
        query = query.filter(BemPatrimonial.regional == regional)
        
    municipio = request.args.get('municipio')
    if municipio:
        query = query.filter(BemPatrimonial.municipio == municipio)
        
    situacao_bem = request.args.get('situacao_bem')
    if situacao_bem:
        query = query.filter(BemPatrimonial.situacao_bem == situacao_bem)
        
    pode_ser_doado = request.args.get('pode_ser_doado')
    if pode_ser_doado:
        query = query.filter(BemPatrimonial.pode_ser_doado == pode_ser_doado)
    
    # Get total records before filtering
    total_records = BemPatrimonial.query.count()
    
    # Apply filtering
    filtered_records = query.count()
    
    # Apply pagination
    bens = query.offset(start).limit(length).all()
    
    data = []
    for bem in bens:
        bem_dict = bem.to_dict()
        bem_dict['acoes'] = f'''
            <div class="btn-group" role="group">
                <button onclick="visualizarDetalhes({bem.id})" class="btn btn-sm btn-info" data-bs-toggle="tooltip" data-bs-placement="top" title="Visualizar detalhes completos deste bem">
                    <i class="fas fa-eye"></i>
                </button>
                <a href="/editar/{bem.id}" class="btn btn-sm btn-warning" data-bs-toggle="tooltip" data-bs-placement="top" title="Editar informações deste bem">
                    <i class="fas fa-edit"></i>
                </a>
                <button onclick="confirmarExclusao({bem.id})" class="btn btn-sm btn-danger" data-bs-toggle="tooltip" data-bs-placement="top" title="Excluir este bem do sistema">
                    <i class="fas fa-trash"></i>
                </button>
            </div>
        '''
        data.append(bem_dict)
    
    return jsonify({
        'draw': draw,
        'recordsTotal': total_records,
        'recordsFiltered': filtered_records,
        'data': data
    })

@app.route('/api/bens/filtros', methods=['GET'])
@login_required
def get_filtros_bens():
    """Retorna valores únicos para os filtros"""
    
    # Obter valores únicos para regionais
    regionais = db.session.query(BemPatrimonial.regional).distinct().order_by(BemPatrimonial.regional).all()
    regionais = [r[0] for r in regionais if r[0]]
    
    # Obter valores únicos para municípios
    municipios = db.session.query(BemPatrimonial.municipio).distinct().order_by(BemPatrimonial.municipio).all()
    municipios = [m[0] for m in municipios if m[0]]
    
    # Obter valores únicos para situações
    situacoes = db.session.query(BemPatrimonial.situacao_bem).distinct().order_by(BemPatrimonial.situacao_bem).all()
    situacoes = [s[0] for s in situacoes if s[0]]
    
    return jsonify({
        'regionais': regionais,
        'municipios': municipios,
        'situacoes': situacoes
    })

@app.route('/api/bens/<int:id>', methods=['GET'])
@login_required
def get_bem_por_id(id):
    """Retorna detalhes de um bem específico por ID"""
    
    bem = BemPatrimonial.query.get_or_404(id)
    
    return jsonify({
        'success': True,
        'bem': bem.to_dict()
    })

@app.route('/cadastro', methods=['GET', 'POST'])
@login_required
def cadastro():
    if request.method == 'POST':
        try:
            # Parsear a data se for fornecida
            data_fiscalizacao = None
            if request.form.get('data_fiscalizacao'):
                data_fiscalizacao = datetime.strptime(request.form.get('data_fiscalizacao'), '%Y-%m-%d')
            
            bem = BemPatrimonial(
                regional=request.form.get('regional'),
                municipio=request.form.get('municipio'),
                comunidade=request.form.get('comunidade'),
                responsavel=request.form.get('responsavel'),
                termo=request.form.get('termo'),
                vigencia=request.form.get('vigencia'),
                item=request.form.get('item'),
                especificacao=request.form.get('especificacao'),
                patrimonio=request.form.get('patrimonio'),
                marca=request.form.get('marca'),
                capacidade=request.form.get('capacidade'),
                placa_serie=request.form.get('placa_serie'),
                pode_ser_doado=request.form.get('pode_ser_doado'),
                situacao_bem=request.form.get('situacao_bem'),
                processo=request.form.get('processo'),
                situacao_processo=request.form.get('situacao_processo'),
                data_fiscalizacao=data_fiscalizacao,
                fiscalizador=request.form.get('fiscalizador')
            )
            
            db.session.add(bem)
            db.session.commit()
            
            flash('Bem patrimonial cadastrado com sucesso!', 'success')
            return redirect(url_for('index'))
        except Exception as e:
            db.session.rollback()
            flash(f'Erro ao cadastrar bem patrimonial: {str(e)}', 'danger')
    
    return render_template('cadastro.html')

@app.route('/editar/<int:id>', methods=['GET', 'POST'])
@login_required
def editar(id):
    bem = BemPatrimonial.query.get_or_404(id)
    
    if request.method == 'POST':
        try:
            # Parsear a data se for fornecida
            data_fiscalizacao = None
            if request.form.get('data_fiscalizacao'):
                data_fiscalizacao = datetime.strptime(request.form.get('data_fiscalizacao'), '%Y-%m-%d')
            
            bem.regional = request.form.get('regional')
            bem.municipio = request.form.get('municipio')
            bem.comunidade = request.form.get('comunidade')
            bem.responsavel = request.form.get('responsavel')
            bem.termo = request.form.get('termo')
            bem.vigencia = request.form.get('vigencia')
            bem.item = request.form.get('item')
            bem.especificacao = request.form.get('especificacao')
            bem.patrimonio = request.form.get('patrimonio')
            bem.marca = request.form.get('marca')
            bem.capacidade = request.form.get('capacidade')
            bem.placa_serie = request.form.get('placa_serie')
            bem.pode_ser_doado = request.form.get('pode_ser_doado')
            bem.situacao_bem = request.form.get('situacao_bem')
            bem.processo = request.form.get('processo')
            bem.situacao_processo = request.form.get('situacao_processo')
            bem.data_fiscalizacao = data_fiscalizacao
            bem.fiscalizador = request.form.get('fiscalizador')
            
            db.session.commit()
            
            flash('Bem patrimonial atualizado com sucesso!', 'success')
            return redirect(url_for('index'))
        except Exception as e:
            db.session.rollback()
            flash(f'Erro ao atualizar bem patrimonial: {str(e)}', 'danger')
    
    return render_template('editar.html', bem=bem)

@app.route('/detalhes/<int:id>')
@login_required
def detalhes(id):
    bem = BemPatrimonial.query.get_or_404(id)
    return render_template('detalhes.html', bem=bem)

@app.route('/excluir/<int:id>', methods=['DELETE'])
@login_required
def excluir(id):
    try:
        bem = BemPatrimonial.query.get_or_404(id)
        db.session.delete(bem)
        db.session.commit()
        return jsonify({'success': True, 'message': 'Bem patrimonial excluído com sucesso!'})
    except Exception as e:
        db.session.rollback()
        return jsonify({'success': False, 'message': f'Erro ao excluir bem patrimonial: {str(e)}'})

@app.route('/dashboard')
@login_required
def dashboard():
    """Exibe o dashboard com infográficos"""
    return render_template('dashboard.html')

@app.route('/api/dashboard/dados', methods=['GET'])
@login_required
def get_dashboard_dados():
    """Retorna dados estatísticos para o dashboard"""
    try:
        # Total de bens cadastrados
        total_bens = BemPatrimonial.query.count()
        
        # Função para tratar valores nulos
        def normalizar_valor(valor):
            """Normaliza valores nulos ou em branco"""
            if valor is None or valor.strip() == '':
                return 'Não informado'
            return valor
            
        # Distribuição por regional
        regionais = db.session.query(
            func.coalesce(BemPatrimonial.regional, 'Não informado').label('regional'), 
            func.count(BemPatrimonial.id).label('total')
        ).group_by('regional').all()
        
        # Garantir que todos os resultados sejam incluídos, mesmo valores nulos
        dados_regionais = {
            'labels': [normalizar_valor(r[0]) for r in regionais],
            'data': [r[1] for r in regionais]
        }
        
        # Distribuição por situação do bem
        situacoes = db.session.query(
            func.coalesce(BemPatrimonial.situacao_bem, 'Não informado').label('situacao'), 
            func.count(BemPatrimonial.id).label('total')
        ).group_by('situacao').all()
        
        dados_situacoes = {
            'labels': [normalizar_valor(s[0]) for s in situacoes],
            'data': [s[1] for s in situacoes]
        }
        
        # Distribuição por município (top 10)
        municipios = db.session.query(
            func.coalesce(BemPatrimonial.municipio, 'Não informado').label('municipio'), 
            func.count(BemPatrimonial.id).label('total')
        ).group_by('municipio').order_by(func.count(BemPatrimonial.id).desc()).limit(10).all()
        
        dados_municipios = {
            'labels': [normalizar_valor(m[0]) for m in municipios],
            'data': [m[1] for m in municipios]
        }
        
        # Distribuição por possibilidade de doação
        doacao = db.session.query(
            func.coalesce(BemPatrimonial.pode_ser_doado, 'Não informado').label('doacao'), 
            func.count(BemPatrimonial.id).label('total')
        ).group_by('doacao').all()
        
        dados_doacao = {
            'labels': [normalizar_valor(d[0]) for d in doacao],
            'data': [d[1] for d in doacao]
        }
        
        # Log para debug dos dados
        app.logger.debug(f"Regionais: {dados_regionais}")
        app.logger.debug(f"Situações: {dados_situacoes}")
        app.logger.debug(f"Municípios: {dados_municipios}")
        app.logger.debug(f"Doação: {dados_doacao}")
        
        return jsonify({
            'success': True,
            'total_bens': total_bens,
            'regionais': dados_regionais,
            'situacoes': dados_situacoes,
            'municipios': dados_municipios,
            'doacao': dados_doacao
        })
    except Exception as e:
        app.logger.error(f"Erro ao gerar dados do dashboard: {str(e)}")
        return jsonify({'success': False, 'error': str(e)})

@app.route('/exportar', methods=['GET'])
@login_required
def exportar():
    try:
        # Get all data
        bens = BemPatrimonial.query.all()
        
        # Create a temporary file
        fd, path = tempfile.mkstemp(suffix='.xlsx')
        os.close(fd)  # Close the file descriptor
        
        # Export data to XLSX
        export_to_xlsx(bens, path)
        
        # Return the file as an attachment
        return send_file(
            path,
            as_attachment=True,
            download_name='bens_patrimoniais.xlsx',
            mimetype='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        )
    except Exception as e:
        flash(f'Erro ao exportar dados: {str(e)}', 'danger')
        return redirect(url_for('index'))

@app.route('/importar', methods=['GET', 'POST'])
@login_required
def importar():
    """Importa dados de uma planilha Excel"""
    if request.method == 'POST':
        try:
            # Verificar se um arquivo foi enviado
            if 'arquivo_excel' not in request.files:
                flash('Nenhum arquivo selecionado', 'danger')
                return redirect(request.url)
                
            arquivo = request.files['arquivo_excel']
            
            # Verificar se o nome do arquivo é vazio
            if arquivo.filename == '':
                flash('Nenhum arquivo selecionado', 'danger')
                return redirect(request.url)
                
            # Verificar se a extensão é .xlsx
            if not arquivo.filename.endswith('.xlsx'):
                flash('Apenas arquivos Excel (.xlsx) são permitidos', 'danger')
                return redirect(request.url)
                
            # Criar um arquivo temporário para o upload
            fd, path = tempfile.mkstemp(suffix='.xlsx')
            os.close(fd)
            
            # Salvar o arquivo enviado
            arquivo.save(path)
            
            # Ler a planilha com pandas
            try:
                df = pd.read_excel(path)
            except Exception as e:
                flash(f'Erro ao ler o arquivo Excel: {str(e)}', 'danger')
                return redirect(request.url)
                
            # Verificar se deve pular o cabeçalho
            pular_cabecalho = request.form.get('pular_cabecalho') == 'on'
            if pular_cabecalho and len(df) > 0:
                df = df.iloc[1:]
                
            # Verificar se há dados para importar
            if len(df) == 0:
                flash('A planilha não contém dados para importar', 'warning')
                return redirect(request.url)
                
            # Colunas esperadas no modelo BemPatrimonial
            colunas_modelo = [
                'regional', 'municipio', 'comunidade', 'responsavel', 'termo', 'vigencia',
                'item', 'especificacao', 'patrimonio', 'marca', 'capacidade', 'placa_serie',
                'pode_ser_doado', 'situacao_bem', 'processo', 'situacao_processo', 
                'data_fiscalizacao', 'fiscalizador'
            ]
            
            # Verificar se as colunas necessárias estão presentes
            colunas_planilha = df.columns.tolist()
            colunas_ausentes = [col for col in colunas_modelo if col not in colunas_planilha]
            
            # Se faltar alguma coluna obrigatória, informar ao usuário
            colunas_obrigatorias = ['regional', 'municipio', 'comunidade', 'responsavel', 'item']
            colunas_obrigatorias_ausentes = [col for col in colunas_obrigatorias if col in colunas_ausentes]
            
            if colunas_obrigatorias_ausentes:
                flash(f'As seguintes colunas obrigatórias estão ausentes: {", ".join(colunas_obrigatorias_ausentes)}', 'danger')
                return redirect(request.url)
            
            # Contadores para o relatório final
            contador_sucesso = 0
            contador_falha = 0
            erros = []
            
            # Percorrer as linhas e importar os dados
            for index, row in df.iterrows():
                try:
                    # Verificar campos obrigatórios
                    if any(pd.isna(row[col]) for col in colunas_obrigatorias if col in colunas_planilha):
                        erros.append(f'Linha {index+1}: Campos obrigatórios não preenchidos')
                        contador_falha += 1
                        continue
                    
                    # Processar a data de fiscalização (se existir)
                    data_fiscalizacao = None
                    if 'data_fiscalizacao' in colunas_planilha and not pd.isna(row['data_fiscalizacao']):
                        try:
                            # Tentar vários formatos de data
                            for formato in ['%d/%m/%Y', '%Y-%m-%d', '%d-%m-%Y', '%m/%d/%Y']:
                                try:
                                    data_fiscalizacao = datetime.strptime(str(row['data_fiscalizacao']), formato)
                                    break
                                except:
                                    continue
                                    
                            # Se ainda não conseguiu parsear, pode ser um número flutuante do Excel
                            if data_fiscalizacao is None and isinstance(row['data_fiscalizacao'], (int, float)):
                                # Converter número do Excel para data Python
                                data_fiscalizacao = datetime.fromordinal(datetime(1900, 1, 1).toordinal() + int(row['data_fiscalizacao']) - 2)
                        except Exception as e:
                            erros.append(f'Linha {index+1}: Erro ao processar data de fiscalização: {str(e)}')
                    
                    # Criar o objeto BemPatrimonial
                    bem = BemPatrimonial(
                        regional=str(row['regional']) if 'regional' in colunas_planilha and not pd.isna(row['regional']) else '',
                        municipio=str(row['municipio']) if 'municipio' in colunas_planilha and not pd.isna(row['municipio']) else '',
                        comunidade=str(row['comunidade']) if 'comunidade' in colunas_planilha and not pd.isna(row['comunidade']) else '',
                        responsavel=str(row['responsavel']) if 'responsavel' in colunas_planilha and not pd.isna(row['responsavel']) else '',
                        termo=str(row['termo']) if 'termo' in colunas_planilha and not pd.isna(row['termo']) else None,
                        vigencia=str(row['vigencia']) if 'vigencia' in colunas_planilha and not pd.isna(row['vigencia']) else None,
                        item=str(row['item']) if 'item' in colunas_planilha and not pd.isna(row['item']) else '',
                        especificacao=str(row['especificacao']) if 'especificacao' in colunas_planilha and not pd.isna(row['especificacao']) else None,
                        patrimonio=str(row['patrimonio']) if 'patrimonio' in colunas_planilha and not pd.isna(row['patrimonio']) else None,
                        marca=str(row['marca']) if 'marca' in colunas_planilha and not pd.isna(row['marca']) else None,
                        capacidade=str(row['capacidade']) if 'capacidade' in colunas_planilha and not pd.isna(row['capacidade']) else None,
                        placa_serie=str(row['placa_serie']) if 'placa_serie' in colunas_planilha and not pd.isna(row['placa_serie']) else None,
                        pode_ser_doado=str(row['pode_ser_doado']) if 'pode_ser_doado' in colunas_planilha and not pd.isna(row['pode_ser_doado']) else None,
                        situacao_bem=str(row['situacao_bem']) if 'situacao_bem' in colunas_planilha and not pd.isna(row['situacao_bem']) else None,
                        processo=str(row['processo']) if 'processo' in colunas_planilha and not pd.isna(row['processo']) else None,
                        situacao_processo=str(row['situacao_processo']) if 'situacao_processo' in colunas_planilha and not pd.isna(row['situacao_processo']) else None,
                        data_fiscalizacao=data_fiscalizacao,
                        fiscalizador=str(row['fiscalizador']) if 'fiscalizador' in colunas_planilha and not pd.isna(row['fiscalizador']) else None
                    )
                    
                    db.session.add(bem)
                    contador_sucesso += 1
                    
                except Exception as e:
                    erros.append(f'Linha {index+1}: {str(e)}')
                    contador_falha += 1
            
            # Commit das alterações ao banco de dados
            if contador_sucesso > 0:
                db.session.commit()
                
                # Mensagem de sucesso
                flash(f'Importação concluída! {contador_sucesso} bens importados com sucesso. {contador_falha} falhas.', 'success')
                
                # Se houver erros, mostrar detalhes
                if erros:
                    flash(f'Ocorreram erros durante a importação em {contador_falha} linhas. Veja os detalhes:', 'warning')
                    for erro in erros[:10]:  # Limitar a 10 erros para não sobrecarregar a página
                        flash(erro, 'warning')
                    if len(erros) > 10:
                        flash(f'... e mais {len(erros) - 10} erros.', 'warning')
                
                return redirect(url_for('index'))
            else:
                db.session.rollback()
                flash('Nenhum bem foi importado devido a erros na planilha. Verifique o formato e os dados.', 'danger')
                
                # Mostrar os primeiros erros
                for erro in erros[:10]:
                    flash(erro, 'danger')
                if len(erros) > 10:
                    flash(f'... e mais {len(erros) - 10} erros.', 'danger')
                
        except Exception as e:
            db.session.rollback()
            flash(f'Erro ao importar dados: {str(e)}', 'danger')
            
        finally:
            # Limpar arquivos temporários
            if 'path' in locals():
                try:
                    os.unlink(path)
                except:
                    pass
    
    return render_template('importar.html')

@app.route('/download-modelo', methods=['GET'])
@login_required
def download_modelo():
    """Gera e faz download de uma planilha modelo para importação"""
    try:
        # Criar um arquivo temporário
        fd, path = tempfile.mkstemp(suffix='.xlsx')
        os.close(fd)
        
        # Criar o modelo
        create_excel_modelo(path)
        
        # Retornar o arquivo para download
        return send_file(
            path,
            as_attachment=True,
            download_name='modelo_importacao_bens.xlsx',
            mimetype='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
        )
    except Exception as e:
        flash(f'Erro ao gerar modelo: {str(e)}', 'danger')
        return redirect(url_for('importar'))

@app.route('/relatorios')
@login_required
def relatorios():
    """Exibe a página de relatórios estatísticos"""
    return render_template('relatorios.html')

@app.route('/api/relatorios/dados', methods=['GET'])
@login_required
def get_relatorios_dados():
    """Retorna dados para os relatórios estatísticos"""
    try:
        tipo_relatorio = request.args.get('tipo', 'situacao')
        periodo = request.args.get('periodo', '12')  # Meses
        
        # Função para normalizar valores nulos
        def normalizar_valor(valor):
            if valor is None or (isinstance(valor, str) and valor.strip() == ''):
                return 'Não informado'
            return valor
            
        if tipo_relatorio == 'situacao':
            # Distribuição por situação do bem
            query = db.session.query(
                func.coalesce(BemPatrimonial.situacao_bem, 'Não informado').label('situacao'),
                func.count(BemPatrimonial.id).label('total')
            ).group_by('situacao').order_by(func.count(BemPatrimonial.id).desc())
            
            resultados = query.all()
            
            dados = {
                'labels': [normalizar_valor(r[0]) for r in resultados],
                'data': [r[1] for r in resultados],
                'titulo': 'Distribuição por Situação do Bem'
            }
            
        elif tipo_relatorio == 'regional':
            # Distribuição por regional
            query = db.session.query(
                func.coalesce(BemPatrimonial.regional, 'Não informado').label('regional'),
                func.count(BemPatrimonial.id).label('total')
            ).group_by('regional').order_by(func.count(BemPatrimonial.id).desc())
            
            resultados = query.all()
            
            dados = {
                'labels': [normalizar_valor(r[0]) for r in resultados],
                'data': [r[1] for r in resultados],
                'titulo': 'Distribuição por Regional'
            }
            
        elif tipo_relatorio == 'municipio':
            # Distribuição por município (top 15)
            query = db.session.query(
                func.coalesce(BemPatrimonial.municipio, 'Não informado').label('municipio'),
                func.count(BemPatrimonial.id).label('total')
            ).group_by('municipio').order_by(func.count(BemPatrimonial.id).desc()).limit(15)
            
            resultados = query.all()
            
            dados = {
                'labels': [normalizar_valor(r[0]) for r in resultados],
                'data': [r[1] for r in resultados],
                'titulo': 'Top 15 Municípios por Quantidade de Bens'
            }
            
        elif tipo_relatorio == 'doacao':
            # Distribuição por possibilidade de doação
            query = db.session.query(
                func.coalesce(BemPatrimonial.pode_ser_doado, 'Não informado').label('doacao'),
                func.count(BemPatrimonial.id).label('total')
            ).group_by('doacao')
            
            resultados = query.all()
            
            dados = {
                'labels': [normalizar_valor(r[0]) for r in resultados],
                'data': [r[1] for r in resultados],
                'titulo': 'Distribuição por Possibilidade de Doação'
            }
            
        elif tipo_relatorio == 'cadastro_temporal':
            # Evolução temporal de cadastros
            # Calcular a data de início baseada no período
            data_inicio = datetime.now().replace(day=1)
            meses = int(periodo)
            for i in range(meses):
                if data_inicio.month == 1:
                    data_inicio = data_inicio.replace(year=data_inicio.year - 1, month=12)
                else:
                    data_inicio = data_inicio.replace(month=data_inicio.month - 1)
            
            # Query para contar bens por mês de cadastro
            from sqlalchemy import extract, and_
            
            resultados = []
            labels = []
            data = []
            
            # Meses desde a data_inicio até agora
            for i in range(meses + 1):
                if i == 0:
                    current_date = data_inicio
                else:
                    if current_date.month == 12:
                        current_date = current_date.replace(year=current_date.year + 1, month=1)
                    else:
                        current_date = current_date.replace(month=current_date.month + 1)
                
                # Formato para label: "Jan/2023"
                month_name = current_date.strftime("%b")
                labels.append(f"{month_name}/{current_date.year}")
                
                count = BemPatrimonial.query.filter(
                    and_(
                        extract('year', BemPatrimonial.data_cadastro) == current_date.year,
                        extract('month', BemPatrimonial.data_cadastro) == current_date.month
                    )
                ).count()
                
                data.append(count)
            
            dados = {
                'labels': labels,
                'data': data,
                'titulo': f'Evolução de Cadastros (Últimos {meses} meses)'
            }
            
        return jsonify({
            'success': True,
            'dados': dados
        })
    except Exception as e:
        app.logger.error(f"Erro ao gerar dados do relatório: {str(e)}")
        return jsonify({'success': False, 'error': str(e)})

@app.route('/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        remember = request.form.get('remember') == 'on'
        
        if not username or not password:
            flash('Por favor, insira nome de usuário e senha.', 'danger')
            return redirect(url_for('login'))
        
        user = User.query.filter_by(username=username).first()
        
        if user and user.check_password(password):
            if not user.is_active:
                flash('Esta conta está desativada. Contate o administrador.', 'danger')
                return redirect(url_for('login'))
                
            login_user(user, remember=remember)
            
            # Atualizar última data de login
            user.last_login = datetime.now()
            db.session.commit()
            
            app.logger.info(f"Usuário {username} autenticado com sucesso")
            
            next_page = request.args.get('next')
            if not next_page or not next_page.startswith('/'):
                next_page = url_for('index')
                
            return redirect(next_page)
        else:
            flash('Nome de usuário ou senha incorretos.', 'danger')
            
    return render_template('login.html')

@app.route('/logout')
@login_required
def logout():
    logout_user()
    flash('Você foi desconectado do sistema.', 'info')
    return redirect(url_for('login'))

@app.route('/admin')
@login_required
def admin():
    if not current_user.is_admin():
        flash('Acesso negado. Você não tem permissão para acessar esta página.', 'danger')
        return redirect(url_for('index'))
        
    # Listar todos os usuários
    users = User.query.all()
    
    return render_template('admin.html', users=users)

@app.route('/add_user', methods=['POST'])
@login_required
def add_user():
    # Verificar se o usuário tem permissão de admin
    if not current_user.is_admin():
        flash('Você não tem permissão para adicionar usuários.', 'danger')
        return redirect(url_for('index'))
    
    try:
        username = request.form.get('username')
        email = request.form.get('email')
        password = request.form.get('password')
        confirm_password = request.form.get('confirm_password')
        role = request.form.get('role')
        
        # Validar entradas
        if not username or not email or not password or not confirm_password or not role:
            flash('Todos os campos são obrigatórios.', 'danger')
            return redirect(url_for('admin'))
            
        # Verificar se as senhas coincidem
        if password != confirm_password:
            flash('As senhas não coincidem.', 'danger')
            return redirect(url_for('admin'))
        
        # Verificar se o usuário já existe
        if User.query.filter_by(username=username).first():
            flash('Nome de usuário já está em uso. Por favor, escolha outro.', 'danger')
            return redirect(url_for('admin'))
            
        # Verificar se o email já existe
        if User.query.filter_by(email=email).first():
            flash('Email já está em uso. Por favor, escolha outro.', 'danger')
            return redirect(url_for('admin'))
            
        # Criar o usuário
        user = User(username=username, email=email, role=role, is_active=True)
        user.set_password(password)
        
        db.session.add(user)
        db.session.commit()
        
        flash(f'Usuário {username} criado com sucesso!', 'success')
        return redirect(url_for('admin'))
        
    except Exception as e:
        db.session.rollback()
        flash(f'Erro ao criar usuário: {str(e)}', 'danger')
        return redirect(url_for('admin'))

@app.route('/edit_user', methods=['POST'])
@login_required
def edit_user():
    # Verificar se o usuário tem permissão de admin
    if not current_user.is_admin():
        flash('Você não tem permissão para editar usuários.', 'danger')
        return redirect(url_for('index'))
    
    try:
        user_id = request.form.get('user_id')
        username = request.form.get('edit_username')
        email = request.form.get('edit_email')
        role = request.form.get('edit_role')
        is_active = 'edit_is_active' in request.form
        
        # Validar entradas
        if not user_id or not username or not email or not role:
            flash('Todos os campos são obrigatórios.', 'danger')
            return redirect(url_for('admin'))
        
        # Obter o usuário
        user = User.query.get_or_404(user_id)
        
        # Verificar nome de usuário duplicado
        existing_user = User.query.filter_by(username=username).first()
        if existing_user and existing_user.id != int(user_id):
            flash('Nome de usuário já está em uso. Por favor, escolha outro.', 'danger')
            return redirect(url_for('admin'))
            
        # Verificar email duplicado
        existing_email = User.query.filter_by(email=email).first()
        if existing_email and existing_email.id != int(user_id):
            flash('Email já está em uso. Por favor, escolha outro.', 'danger')
            return redirect(url_for('admin'))
        
        # Atualizar o usuário
        user.username = username
        user.email = email
        user.role = role
        user.is_active = is_active
        
        db.session.commit()
        
        flash(f'Usuário {username} atualizado com sucesso!', 'success')
        return redirect(url_for('admin'))
        
    except Exception as e:
        db.session.rollback()
        flash(f'Erro ao atualizar usuário: {str(e)}', 'danger')
        return redirect(url_for('admin'))

@app.route('/delete_user', methods=['POST'])
@login_required
def delete_user():
    # Verificar se o usuário tem permissão de admin
    if not current_user.is_admin():
        flash('Você não tem permissão para excluir usuários.', 'danger')
        return redirect(url_for('index'))
    
    try:
        user_id = request.form.get('user_id')
        
        # Não permitir que o usuário exclua a si mesmo
        if int(user_id) == current_user.id:
            flash('Você não pode excluir seu próprio usuário!', 'danger')
            return redirect(url_for('admin'))
        
        # Obter o usuário
        user = User.query.get_or_404(user_id)
        
        # Armazenar o nome para a mensagem
        username = user.username
        
        # Excluir o usuário
        db.session.delete(user)
        db.session.commit()
        
        flash(f'Usuário {username} excluído com sucesso!', 'success')
        return redirect(url_for('admin'))
        
    except Exception as e:
        db.session.rollback()
        flash(f'Erro ao excluir usuário: {str(e)}', 'danger')
        return redirect(url_for('admin'))

@app.route('/reset_password', methods=['POST'])
@login_required
def reset_password():
    # Verificar se o usuário tem permissão de admin
    if not current_user.is_admin() and int(request.form.get('user_id')) != current_user.id:
        flash('Você não tem permissão para resetar senhas.', 'danger')
        return redirect(url_for('index'))
    
    try:
        user_id = request.form.get('user_id')
        new_password = request.form.get('new_password')
        confirm_new_password = request.form.get('confirm_new_password')
        
        # Validar entradas
        if not user_id or not new_password or not confirm_new_password:
            flash('Todos os campos são obrigatórios.', 'danger')
            return redirect(url_for('admin'))
            
        # Verificar se as senhas coincidem
        if new_password != confirm_new_password:
            flash('As senhas não coincidem.', 'danger')
            return redirect(url_for('admin'))
        
        # Obter o usuário
        user = User.query.get_or_404(user_id)
        
        # Resetar senha
        user.set_password(new_password)
        db.session.commit()
        
        flash(f'Senha do usuário {user.username} alterada com sucesso!', 'success')
        
        # Se foi o próprio usuário que alterou a senha, redirecionar para o logout
        if int(user_id) == current_user.id:
            return redirect(url_for('logout'))
            
        return redirect(url_for('admin'))
        
    except Exception as e:
        db.session.rollback()
        flash(f'Erro ao resetar senha: {str(e)}', 'danger')
        return redirect(url_for('admin'))

@app.route('/esqueci-senha', methods=['GET', 'POST'])
def esqueci_senha():
    if current_user.is_authenticated:
        return redirect(url_for('index'))
        
    if request.method == 'POST':
        username = request.form.get('username')
        email = request.form.get('email')
        
        if not username or not email:
            flash('Por favor, preencha todos os campos.', 'danger')
            return render_template('esqueci_senha.html')
            
        # Verificar se o usuário existe e se o email corresponde
        user = User.query.filter_by(username=username, email=email).first()
        
        if not user:
            flash('Não foi encontrado um usuário com essas credenciais.', 'danger')
            return render_template('esqueci_senha.html')
            
        # Gerar senha temporária
        temp_password = ''.join([str(random.randint(0, 9)) for _ in range(6)])
        
        # Atualizar senha do usuário
        user.set_password(temp_password)
        db.session.commit()
        
        # Em um sistema real, aqui enviaríamos um email com a senha temporária
        # Como estamos em um ambiente de demonstração, exibimos a senha na tela
        flash(f'Senha temporária gerada com sucesso: {temp_password}. Faça login com essa senha e altere-a imediatamente.', 'success')
        return redirect(url_for('login'))
        
    return render_template('esqueci_senha.html')

@app.route('/get_user/<int:id>', methods=['GET'])
@login_required
def get_user(id):
    # Verificar se o usuário tem permissão de admin
    if not current_user.is_admin():
        return jsonify({'success': False, 'message': 'Acesso negado'})
    
    try:
        user = User.query.get_or_404(id)
        
        return jsonify({
            'success': True,
            'user': {
                'id': user.id,
                'username': user.username,
                'email': user.email,
                'role': user.role,
                'is_active': user.is_active
            }
        })
    except Exception as e:
        return jsonify({'success': False, 'message': str(e)})


import pandas as pd
import openpyxl
from openpyxl.styles import Font, Alignment, PatternFill, Border, Side, NamedStyle
from openpyxl.utils import get_column_letter

def export_to_xlsx(bens, output_path):
    """
    Export bens (assets) to an XLSX file.
    
    Args:
        bens: List of BemPatrimonial objects
        output_path: Path to save the output file
    """
    # Criar um dataframe com os dados
    data = []
    for bem in bens:
        data.append({
            'ID': bem.id,
            'Regional': bem.regional,
            'Município': bem.municipio,
            'Comunidade': bem.comunidade,
            'Responsável': bem.responsavel,
            'Termo': bem.termo,
            'Vigência': bem.vigencia,
            'Item': bem.item,
            'Especificação': bem.especificacao,
            'Patrimônio': bem.patrimonio,
            'Marca': bem.marca,
            'Capacidade': bem.capacidade,
            'Placa/Série': bem.placa_serie,
            'Já pode ser doado?': bem.pode_ser_doado,
            'Situação do bem': bem.situacao_bem,
            'Processo': bem.processo,
            'Situação do Processo': bem.situacao_processo,
            'Data da Fiscalização': bem.data_fiscalizacao.strftime('%d/%m/%Y') if bem.data_fiscalizacao else '',
            'Fiscalizador': bem.fiscalizador,
            'Data de Cadastro': bem.data_cadastro.strftime('%d/%m/%Y %H:%M')
        })
    
    # Criar dataframe
    df = pd.DataFrame(data)
    
    # Exportar para Excel
    df.to_excel(output_path, index=False, sheet_name='Bens Patrimoniais')
    
    # Formatar o arquivo Excel
    wb = openpyxl.load_workbook(output_path)
    ws = wb.active
    
    # Estilo para o cabeçalho
    header_font = Font(bold=True, color="FFFFFF")
    header_fill = PatternFill(start_color="1F4E78", end_color="1F4E78", fill_type="solid")
    header_alignment = Alignment(horizontal='center', vertical='center', wrap_text=True)
    
    # Aplicar estilos ao cabeçalho
    for cell in ws[1]:
        cell.font = header_font
        cell.fill = header_fill
        cell.alignment = header_alignment
    
    # Ajustar largura das colunas
    for column in ws.columns:
        max_length = 0
        column_letter = column[0].column_letter
        
        for cell in column:
            if cell.value:
                try:
                    if len(str(cell.value)) > max_length:
                        max_length = len(str(cell.value))
                except:
                    pass
        
        adjusted_width = max_length + 2
        ws.column_dimensions[column_letter].width = min(adjusted_width, 50)
    
    # Bordas para todas as células
    border = Border(
        left=Side(style='thin'), 
        right=Side(style='thin'), 
        top=Side(style='thin'), 
        bottom=Side(style='thin')
    )
    
    for row in ws.iter_rows(min_row=1, max_row=ws.max_row, min_col=1, max_col=ws.max_column):
        for cell in row:
            cell.border = border
            
            # Centralizar algumas colunas
            if cell.column in [1, 14, 18]:  # ID, Já pode ser doado?, Data da Fiscalização
                cell.alignment = Alignment(horizontal='center')
    
    # Salvar as alterações
    wb.save(output_path)
    
def create_excel_modelo(output_path):
    """
    Cria um arquivo Excel modelo para importação de bens patrimoniais.
    
    Args:
        output_path: Caminho onde o arquivo modelo será salvo
    """
    # Criar um workbook vazio
    wb = openpyxl.Workbook()
    ws = wb.active
    ws.title = "Modelo de Importação"
    
    # Definir os cabeçalhos de acordo com o modelo de dados
    cabecalhos = [
        'regional', 
        'municipio', 
        'comunidade', 
        'responsavel', 
        'termo', 
        'vigencia',
        'item', 
        'especificacao', 
        'patrimonio', 
        'marca', 
        'capacidade', 
        'placa_serie',
        'pode_ser_doado', 
        'situacao_bem', 
        'processo', 
        'situacao_processo',
        'data_fiscalizacao', 
        'fiscalizador'
    ]
    
    # Adicionar cabeçalhos
    for col_num, cabecalho in enumerate(cabecalhos, 1):
        ws.cell(row=1, column=col_num, value=cabecalho)
    
    # Estilos
    # Cabeçalho
    header_style = NamedStyle(name="header_style")
    header_style.font = Font(bold=True, color="FFFFFF")
    header_style.fill = PatternFill(start_color="1F4E78", end_color="1F4E78", fill_type="solid")
    header_style.alignment = Alignment(horizontal='center', vertical='center', wrap_text=True)
    header_style.border = Border(
        left=Side(style='thin'), 
        right=Side(style='thin'), 
        top=Side(style='thin'), 
        bottom=Side(style='thin')
    )
    
    # Aplicar estilo ao cabeçalho
    for cell in ws[1]:
        cell.style = header_style
    
    # Adicionar alguns exemplos para ajudar
    exemplos = [
        {
            'regional': 'Norte',
            'municipio': 'Manaus',
            'comunidade': 'Comunidade Nova Esperança',
            'responsavel': 'João Silva',
            'termo': 'TER-123/2020',
            'vigencia': '31/12/2025',
            'item': 'Computador',
            'especificacao': 'Desktop i5, 8GB RAM, 500GB HD',
            'patrimonio': 'PATR-12345',
            'marca': 'Dell',
            'capacidade': 'N/A',
            'placa_serie': 'XYZ789',
            'pode_ser_doado': 'Sim',
            'situacao_bem': 'Em uso',
            'processo': 'PROC-456/2020',
            'situacao_processo': 'Concluído',
            'data_fiscalizacao': '15/01/2023',
            'fiscalizador': 'Maria Oliveira'
        },
        {
            'regional': 'Sul',
            'municipio': 'Porto Alegre',
            'comunidade': 'Comunidade Harmonia',
            'responsavel': 'Pedro Santos',
            'termo': 'TER-456/2021',
            'vigencia': '30/06/2026',
            'item': 'Impressora',
            'especificacao': 'Multifuncional jato de tinta colorida',
            'patrimonio': 'PATR-67890',
            'marca': 'Epson',
            'capacidade': 'N/A',
            'placa_serie': 'ABC123',
            'pode_ser_doado': 'Não',
            'situacao_bem': 'Em uso',
            'processo': 'PROC-789/2021',
            'situacao_processo': 'Em andamento',
            'data_fiscalizacao': '20/02/2024',
            'fiscalizador': 'Carlos Ferreira'
        }
    ]
    
    # Adicionar exemplos como linhas
    for row_idx, exemplo in enumerate(exemplos, 2):
        for col_idx, campo in enumerate(cabecalhos, 1):
            ws.cell(row=row_idx, column=col_idx, value=exemplo.get(campo, ''))
    
    # Largura das colunas
    for idx, cabecalho in enumerate(cabecalhos, 1):
        col_letter = get_column_letter(idx)
        # Definir largura baseada no comprimento do cabeçalho
        ws.column_dimensions[col_letter].width = max(len(cabecalho) + 5, 15)
    
    # Adicionar uma instrução na célula A4
    ws.cell(row=4, column=1, value="ATENÇÃO: Preencha seus dados abaixo desta linha. Os exemplos acima são apenas para referência.")
    ws.merge_cells(start_row=4, start_column=1, end_row=4, end_column=len(cabecalhos))
    
    # Estilo da célula de instrução
    instrucao_cell = ws.cell(row=4, column=1)
    instrucao_cell.font = Font(bold=True, color="FF0000")
    instrucao_cell.alignment = Alignment(horizontal='center')
    
    # Adicionar algumas linhas vazias para preenchimento
    for i in range(5, 10):
        for j in range(1, len(cabecalhos) + 1):
            ws.cell(row=i, column=j, value="")
    
    # Congelando painel para facilitar a visualização
    ws.freeze_panes = "A2"
    
    # Salvar o arquivo
    wb.save(output_path)

from app import app, create_admin_user
import routes  # Import routes to register them

if __name__ == "__main__":
    # Criar usuário administrador padrão (apenas para desenvolvimento)
    create_admin_user('admin', 'admin@sgpb.com', 'admin123')
    
    app.run(host="0.0.0.0", port=5000, debug=True)


from app import app, db, create_admin_user
from models import User

with app.app_context():
    print("Verificando usuários no banco de dados...")
    users = User.query.all()
    if users:
        print(f"Usuários encontrados: {len(users)}")
        for user in users:
            print(f"ID: {user.id}, Username: {user.username}, Email: {user.email}, Role: {user.role}")
    else:
        print("Nenhum usuário encontrado no banco de dados.")
        print("Criando usuário administrador...")
        success = create_admin_user('admin', 'admin@sgpb.com', 'admin123')
        print(f"Criação de usuário admin: {'Sucesso' if success else 'Falha'}")

{% extends 'layout.html' %}

{% block content %}
<div class="row justify-content-center mt-5">
    <div class="col-md-6">
        <div class="card shadow">
            <div class="card-header bg-primary text-white">
                <h5 class="mb-0"><i class="fas fa-key me-2"></i>Recuperação de Senha</h5>
            </div>
            <div class="card-body">
                {% with messages = get_flashed_messages(with_categories=true) %}
                    {% if messages %}
                        {% for category, message in messages %}
                            <div class="alert alert-{{ category }} alert-dismissible fade show" role="alert">
                                {{ message }}
                                <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
                            </div>
                        {% endfor %}
                    {% endif %}
                {% endwith %}
                
                <p class="alert alert-info">
                    <i class="fas fa-info-circle me-2"></i>Informe seu nome de usuário e email para gerar uma senha temporária.
                </p>
                
                <form method="POST" action="{{ url_for('esqueci_senha') }}">
                    <div class="mb-3">
                        <label for="username" class="form-label">Nome de Usuário</label>
                        <div class="input-group">
                            <span class="input-group-text"><i class="fas fa-user"></i></span>
                            <input type="text" class="form-control" id="username" name="username" required autofocus>
                        </div>
                    </div>
                    <div class="mb-3">
                        <label for="email" class="form-label">Email</label>
                        <div class="input-group">
                            <span class="input-group-text"><i class="fas fa-envelope"></i></span>
                            <input type="email" class="form-control" id="email" name="email" required>
                        </div>
                    </div>
                    <div class="d-grid gap-2">
                        <button type="submit" class="btn btn-primary btn-lg">
                            <i class="fas fa-paper-plane me-2"></i>Solicitar Nova Senha
                        </button>
                    </div>
                </form>
                
                <div class="mt-3 text-center">
                    <a href="{{ url_for('login') }}" class="text-muted">Voltar para o login</a>
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}

{% extends 'layout.html' %}

{% block content %}
<div class="row justify-content-center mt-5">
    <div class="col-md-6">
        <div class="card shadow">
            <div class="card-header bg-primary text-white">
                <h5 class="mb-0"><i class="fas fa-sign-in-alt me-2"></i>Login - Gerenciamento de Bens</h5>
            </div>
            <div class="card-body">
                {% with messages = get_flashed_messages(with_categories=true) %}
                    {% if messages %}
                        {% for category, message in messages %}
                            <div class="alert alert-{{ category }} alert-dismissible fade show" role="alert">
                                {{ message }}
                                <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
                            </div>
                        {% endfor %}
                    {% endif %}
                {% endwith %}
                
                <form method="POST" action="{{ url_for('login') }}">
                    <div class="mb-3">
                        <label for="username" class="form-label">Nome de Usuário</label>
                        <div class="input-group">
                            <span class="input-group-text"><i class="fas fa-user"></i></span>
                            <input type="text" class="form-control" id="username" name="username" required autofocus>
                        </div>
                    </div>
                    <div class="mb-3">
                        <label for="password" class="form-label">Senha</label>
                        <div class="input-group">
                            <span class="input-group-text"><i class="fas fa-lock"></i></span>
                            <input type="password" class="form-control" id="password" name="password" required>
                        </div>
                    </div>
                    <div class="mb-3 form-check">
                        <input type="checkbox" class="form-check-input" id="remember" name="remember">
                        <label class="form-check-label" for="remember">Lembrar-me</label>
                    </div>
                    <div class="d-grid gap-2">
                        <button type="submit" class="btn btn-primary btn-lg">
                            <i class="fas fa-sign-in-alt me-2"></i>Entrar
                        </button>
                    </div>
                    <div class="mt-3 text-center">
                        <a href="{{ url_for('esqueci_senha') }}" class="text-muted">
                            <i class="fas fa-question-circle me-1"></i>Esqueceu sua senha?
                        </a>
                    </div>
                </form>
            </div>
            <div class="card-footer bg-light">
                <p class="text-center text-muted mb-0">
                    Acesso restrito a usuários autorizados
                </p>
            </div>
        </div>
    </div>
</div>
{% endblock %}




