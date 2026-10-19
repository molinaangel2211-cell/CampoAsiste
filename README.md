import React, { useState, useEffect, createContext, useContext } from 'react';

// ==========================================
// 1. TIPOS DE DATOS Y CONTRATOS
// ==========================================

export type RolUsuario = 'agricultor' | 'transportador' | 'entidad' | 'admin';

export type EstadoFinanciacion = 
  | 'Borrador'
  | 'Pendiente de revisión'
  | 'En evaluación'
  | 'Requiere información adicional'
  | 'Aprobada por la entidad'
  | 'Rechazada por la entidad'
  | 'Cancelada por el agricultor';

export type EstadoTransporte = 
  | 'Abierta'
  | 'Recibiendo propuestas'
  | 'Transporte acordado'
  | 'En tránsito'
  | 'Entrega completada'
  | 'Cancelada';

export interface Usuario {
  id: string;
  nombre: string;
  correo: string;
  telefono: string;
  departamento: string;
  municipio: string;
  rol: RolUsuario;
}

export interface SolicitudFinanciacion {
  id: string;
  agricultorId: string;
  agricultorNombre: string;
  cultivo: string;
  departamento: string;
  municipio: string;
  montoCop: number;
  areaHectareas: number;
  finalidad: 'Semillas' | 'Fertilizantes' | 'Maquinaria' | 'Adecuación de tierra' | 'Insumos varios';
  descripcion?: string;
  fechaRequerida: string;
  entidadPreferida: string;
  estado: EstadoFinanciacion;
  fechaCreacion: string;
  historial: { fecha: string; estado: EstadoFinanciacion; nota: string; autor: string }[];
}

export interface PropuestaFlete {
  id: string;
  transporteId: string;
  transportadorId: string;
  transportadorNombre: string;
  telefono: string;
  vehiculo: string;
  placa: string;
  precioCop: number;
  mensaje: string;
  fecha: string;
}

export interface PublicacionTransporte {
  id: string;
  agricultorId: string;
  agricultorNombre: string;
  producto: string;
  cantidadValor: number;
  unidad: 'kg' | 'toneladas' | 'cargas' | 'bultos';
  origenDepto: string;
  origenMunicipio: string;
  veredaDetallada: string; // Oculta hasta acordar servicio
  destinoDepto: string;
  destinoMunicipio: string;
  fechaRecogida: string;
  tipoVehiculoSugerido: string;
  pagoOfrecidoCop: number | 'Negociable';
  estado: EstadoTransporte;
  telefonoContacto: string; // Oculto hasta acordar servicio
  propuestas: PropuestaFlete[];
  transportadorAcordadoId?: string;
}

// ==========================================
// 2. DATOS DE COLOMBIA Y SEMILLAS DEMO
// ==========================================

const MUNICIPIOS_COLOMBIA: Record<string, string[]> = {
  'Boyacá': ['Tunja', 'Ventaquemada', 'Sogamoso', 'Duitama', 'Chiquinquirá', 'Villa de Leyva'],
  'Cundinamarca': ['Facatativá', 'Fusagasugá', 'Zipaquirá', 'Ubaté', 'Cáqueza', 'Bogotá D.C.'],
  'Antioquia': ['Rionegro', 'Marinilla', 'Santa Rosa de Osos', 'Andes', 'Urrao'],
  'Nariño': ['Pasto', 'Ipiales', 'Túquerres', 'La Unión', 'Sandoná'],
  'Huila': ['Pitalito', 'Garzón', 'La Plata', 'Neiva'],
  'Tolima': ['Ibagué', 'Espinal', 'Líbano', 'Chaparral']
};

const USUARIOS_DEMO: Usuario[] = [
  { id: 'usr-1', nombre: 'José Alirio Pérez', correo: 'jose@campo.co', telefono: '3110001122', departamento: 'Boyacá', municipio: 'Ventaquemada', rol: 'agricultor' },
  { id: 'usr-2', nombre: 'Marcos Camargo', correo: 'marcos@fletes.co', telefono: '3204445566', departamento: 'Boyacá', municipio: 'Tunja', rol: 'transportador' },
  { id: 'usr-3', nombre: 'Oficial de Crédito Rural', correo: 'credito@coopboyaca.co', telefono: '3157778899', departamento: 'Boyacá', municipio: 'Tunja', rol: 'entidad' },
  { id: 'usr-4', nombre: 'Supervisora Plataforma', correo: 'admin@campoasiste.co', telefono: '3001112233', departamento: 'Cundinamarca', municipio: 'Bogotá D.C.', rol: 'admin' }
];

const SOLICITUD_INICIAL: SolicitudFinanciacion = {
  id: 'FIN-2026-101',
  agricultorId: 'usr-1',
  agricultorNombre: 'José Alirio Pérez',
  cultivo: 'Papa Pastusa',
  departamento: 'Boyacá',
  municipio: 'Ventaquemada',
  montoCop: 6500000,
  areaHectareas: 2.0,
  finalidad: 'Fertilizantes',
  descripcion: 'Fertilización inicial y control fitosanitario de siembra semestral.',
  fechaRequerida: '2026-05-10',
  entidadPreferida: 'Coop Agrocredit Boyacá (Demostración)',
  estado: 'En evaluación',
  fechaCreacion: '2026-04-01',
  historial: [
    { fecha: '2026-04-01', estado: 'Pendiente de revisión', nota: 'Radicado digital por el agricultor.', autor: 'José Alirio Pérez' },
    { fecha: '2026-04-03', estado: 'En evaluación', nota: 'Revisión técnica de capacidad de hectáreas.', autor: 'Oficial de Crédito Rural' }
  ]
};

const TRANSPORTE_INICIAL: PublicacionTransporte = {
  id: 'TRP-2026-501',
  agricultorId: 'usr-1',
  agricultorNombre: 'José Alirio Pérez',
  producto: 'Papa Superior seleccionada',
  cantidadValor: 120,
  unidad: 'bultos',
  origenDepto: 'Boyacá',
  origenMunicipio: 'Ventaquemada',
  veredaDetallada: 'Vereda El Carmen, km 4 vía secundaria',
  destinoDepto: 'Cundinamarca',
  destinoMunicipio: 'Bogotá D.C.',
  fechaRecogida: '2026-05-02',
  tipoVehiculoSugerido: 'Camión Turbo NPR',
  pagoOfrecidoCop: 1500000,
  estado: 'Recibiendo propuestas',
  telefonoContacto: '3110001122',
  propuestas: [
    {
      id: 'PROP-1',
      transporteId: 'TRP-2026-501',
      transportadorId: 'usr-2',
      transportadorNombre: 'Marcos Camargo',
      telefono: '3204445566',
      vehiculo: 'Chevrolet NPR Carpado',
      placa: 'SOJ-789',
      precioCop: 1450000,
      mensaje: 'Disponible para cargar a las 6:00 a.m. Cuento con carpa impermeable.',
      fecha: '2026-04-05'
    }
  ]
};

// ==========================================
// 3. CONTEXTO DE AUTENTICACIÓN
// ==========================================

interface AuthContextType {
  usuarioActual: Usuario | null;
  iniciarSesion: (correo: string) => boolean;
  registrarse: (usuario: Omit<Usuario, 'id'>) => void;
  cerrarSesion: () => void;
}

const AuthContext = createContext<AuthContextType>({
  usuarioActual: null,
  iniciarSesion: () => false,
  registrarse: () => {},
  cerrarSesion: () => {},
});

export const useAuth = () => useContext(AuthContext);

// ==========================================
// 4. COMPONENTE PRINCIPAL
// ==========================================

export default function CampoAsisteApp() {
  const [usuarios, setUsuarios] = useState<Usuario[]>(() => {
    const loc = localStorage.getItem('ca_usuarios_v2');
    return loc ? JSON.parse(loc) : USUARIOS_DEMO;
  });

  const [usuarioActual, setUsuarioActual] = useState<Usuario | null>(() => {
    const loc = localStorage.getItem('ca_sesion_v2');
    return loc ? JSON.parse(loc) : USUARIOS_DEMO[0]; // Inicia con el agricultor demo
  });

  const [pantalla, setPantalla] = useState<'inicio' | 'financiacion' | 'transporte' | 'banco' | 'admin' | 'perfil'>('inicio');
  const [conectado, setConectado] = useState<boolean>(navigator.onLine);

  // Estados de datos
  const [solicitudes, setSolicitudes] = useState<SolicitudFinanciacion[]>(() => {
    const loc = localStorage.getItem('ca_solicitudes_v2');
    return loc ? JSON.parse(loc) : [SOLICITUD_INICIAL];
  });

  const [transportes, setTransportes] = useState<PublicacionTransporte[]>(() => {
    const loc = localStorage.getItem('ca_transportes_v2');
    return loc ? JSON.parse(loc) : [TRANSPORTE_INICIAL];
  });

  const [alerta, setAlerta] = useState<{ texto: string; tipo: 'exito' | 'error' | 'info' } | null>(null);

  // Detectar conectividad en el teléfono
  useEffect(() => {
    const online = () => setConectado(true);
    const offline = () => setConectado(false);
    window.addEventListener('online', online);
    window.addEventListener('offline', offline);
    return () => {
      window.removeEventListener('online', online);
      window.removeEventListener('offline', offline);
    };
  }, []);

  // Persistencia local
  useEffect(() => {
    localStorage.setItem('ca_usuarios_v2', JSON.stringify(usuarios));
  }, [usuarios]);

  useEffect(() => {
    localStorage.setItem('ca_solicitudes_v2', JSON.stringify(solicitudes));
  }, [solicitudes]);

  useEffect(() => {
    localStorage.setItem('ca_transportes_v2', JSON.stringify(transportes));
  }, [transportes]);

  useEffect(() => {
    if (usuarioActual) {
      localStorage.setItem('ca_sesion_v2', JSON.stringify(usuarioActual));
    } else {
      localStorage.removeItem('ca_sesion_v2');
    }
  }, [usuarioActual]);

  const notificar = (texto: string, tipo: 'exito' | 'error' | 'info' = 'exito') => {
    setAlerta({ texto, tipo });
    setTimeout(() => setAlerta(null), 4000);
  };

  const iniciarSesion = (correo: string): boolean => {
    const u = usuarios.find(user => user.correo.toLowerCase() === correo.trim().toLowerCase());
    if (u) {
      setUsuarioActual(u);
      notificar(`Bienvenido de nuevo, ${u.nombre}`, 'exito');
      return true;
    }
    notificar('No se encontró una cuenta con ese correo', 'error');
    return false;
  };

  const registrarse = (datos: Omit<Usuario, 'id'>) => {
    const nuevoUsuario: Usuario = {
      ...datos,
      id: `usr-${Date.now()}`
    };
    setUsuarios(prev => [...prev, nuevoUsuario]);
    setUsuarioActual(nuevoUsuario);
    notificar('Cuenta creada correctamente', 'exito');
  };

  const cerrarSesion = () => {
    setUsuarioActual(null);
    setPantalla('inicio');
    notificar('Sesión cerrada', 'info');
  };

  const formatoCOP = (monto: number) => {
    return new Intl.NumberFormat('es-CO', { style: 'currency', currency: 'COP', maximumFractionDigits: 0 }).format(monto);
  };

  return (
    <AuthContext.Provider value={{ usuarioActual, iniciarSesion, registrarse, cerrarSesion }}>
      <div style={estilos.contenedorApp}>
        {/* Aviso de conexión móvil */}
        {!conectado && (
          <div style={estilos.bannerSinRed}>
            ⚠️ Modo sin internet rural: la información se guarda solo en este teléfono.
          </div>
        )}

        {/* Mensajes flotantes */}
        {alerta && (
          <div style={{
            ...estilos.alertaFlotante,
            backgroundColor: alerta.tipo === 'exito' ? '#1B4332' : alerta.tipo === 'error' ? '#8B0000' : '#2D6A4F'
          }}>
            {alerta.texto}
          </div>
        )}

        {/* Encabezado Superior */}
        <header style={estilos.cabecera}>
          <div style={estilos.cabeceraSuperior}>
            <div style={estilos.logoIcono}>🌱</div>
            <div>
              <h1 style={estilos.tituloLogo}>CampoAsiste</h1>
              <p style={estilos.eslogan}>Conectando el campo con nuevas oportunidades</p>
            </div>
          </div>

          {/* Estado de sesión actual */}
          <div style={estilos.barraUsuario}>
            {usuarioActual ? (
              <div style={estilos.usuarioInfoRow}>
                <span>👤 <strong>{usuarioActual.nombre}</strong> ({usuarioActual.rol.toUpperCase()})</span>
                <button onClick={cerrarSesion} style={estilos.botonCerrarSesion}>Salir</button>
              </div>
            ) : (
              <span style={estilos.textoInvitado}>Modo visitante: inicie sesión para publicar o pedir crédito</span>
            )}
          </div>
        </header>

        {/* Contenido según la pantalla seleccionada */}
        <main style={estilos.cuerpoPrincipal}>
          {pantalla === 'inicio' && (
            <PantallaInicio 
              irA={(p) => setPantalla(p)} 
              totalCreditos={solicitudes.length} 
              totalFletes={transportes.length} 
            />
          )}

          {pantalla === 'financiacion' && (
            <PantallaFinanciacion 
              solicitudes={solicitudes} 
              usuario={usuarioActual}
              formatoCOP={formatoCOP}
              onCrear={(nueva) => {
                setSolicitudes([nueva, ...solicitudes]);
                notificar('Solicitud radicada. En espera de evaluación de la entidad.', 'exito');
              }}
            />
          )}

          {pantalla === 'transporte' && (
            <PantallaTransporte 
              transportes={transportes}
              usuario={usuarioActual}
              formatoCOP={formatoCOP}
              onCrearCarga={(carga) => {
                setTransportes([carga, ...transportes]);
                notificar('Carga publicada para los transportadores.', 'exito');
              }}
              onEnviarPropuesta={(propuesta) => {
                setTransportes(prev => prev.map(t => {
                  if (t.id === propuesta.transporteId) {
                    return { ...t, propuestas: [...t.propuestas, propuesta], estado: 'Recibiendo propuestas' };
                  }
                  return t;
                }));
                notificar('Propuesta de transporte enviada al agricultor.', 'exito');
              }}
              onAceptarPropuesta={(transporteId, propuesta) => {
                setTransportes(prev => prev.map(t => {
                  if (t.id === transporteId) {
                    return { ...t, estado: 'Transporte acordado', transportadorAcordadoId: propuesta.transportadorId };
                  }
                  return t;
                }));
                notificar(`Flete acordado con ${propuesta.transportadorNombre}. Teléfonos desbloqueados.`, 'exito');
              }}
            />
          )}

          {pantalla === 'banco' && (
            <PantallaBanco 
              solicitudes={solicitudes}
              usuario={usuarioActual}
              formatoCOP={formatoCOP}
              onActualizar={(id, nuevoEstado, nota) => {
                setSolicitudes(prev => prev.map(s => {
                  if (s.id === id) {
                    return {
                      ...s,
                      estado: nuevoEstado,
                      historial: [
                        ...s.historial,
                        {
                          fecha: new Date().toISOString().split('T')[0],
                          estado: nuevoEstado,
                          nota: nota || 'Actualización de estudio crediticio.',
                          autor: usuarioActual?.nombre || 'Oficial de crédito'
                        }
                      ]
                    };
                  }
                  return s;
                }));
                notificar('Decisión de crédito guardada.', 'exito');
              }}
            />
          )}

          {pantalla === 'admin' && (
            <PantallaAdmin 
              solicitudes={solicitudes} 
              transportes={transportes} 
              usuarios={usuarios} 
              usuario={usuarioActual}
              onEliminarTransporte={(id) => {
                setTransportes(prev => prev.filter(t => t.id !== id));
                notificar('Publicación eliminada por moderación', 'info');
              }}
            />
          )}

          {pantalla === 'perfil' && (
            <PantallaPerfil 
              usuario={usuarioActual} 
              usuariosDemo={usuarios} 
              onCambiarSesionRapida={(correo) => iniciarSesion(correo)} 
            />
          )}
        </main>

        {/* Barra de Navegación Inferior (Touch-friendly para Android) */}
        <nav style={estilos.barraInferior}>
          <button onClick={() => setPantalla('inicio')} style={{ ...estilos.itemNav, color: pantalla === 'inicio' ? '#2D6A4F' : '#6B7280' }}>
            <span style={estilos.iconoNav}>🌾</span>
            <span style={estilos.textoNav}>Inicio</span>
          </button>

          <button onClick={() => setPantalla('financiacion')} style={{ ...estilos.itemNav, color: pantalla === 'financiacion' ? '#2D6A4F' : '#6B7280' }}>
            <span style={estilos.iconoNav}>💰</span>
            <span style={estilos.textoNav}>Créditos</span>
          </button>

          <button onClick={() => setPantalla('transporte')} style={{ ...estilos.itemNav, color: pantalla === 'transporte' ? '#2D6A4F' : '#6B7280' }}>
            <span style={estilos.iconoNav}>🚛</span>
            <span style={estilos.textoNav}>Transporte</span>
          </button>

          {usuarioActual?.rol === 'entidad' && (
            <button onClick={() => setPantalla('banco')} style={{ ...estilos.itemNav, color: pantalla === 'banco' ? '#2D6A4F' : '#6B7280' }}>
              <span style={estilos.iconoNav}>🏛️</span>
              <span style={estilos.textoNav}>Evaluación</span>
            </button>
          )}

          {usuarioActual?.rol === 'admin' && (
            <button onClick={() => setPantalla('admin')} style={{ ...estilos.itemNav, color: pantalla === 'admin' ? '#2D6A4F' : '#6B7280' }}>
              <span style={estilos.iconoNav}>🛡️</span>
              <span style={estilos.textoNav}>Control</span>
            </button>
          )}

          <button onClick={() => setPantalla('perfil')} style={{ ...estilos.itemNav, color: pantalla === 'perfil' ? '#2D6A4F' : '#6B7280' }}>
            <span style={estilos.iconoNav}>👤</span>
            <span style={estilos.textoNav}>Cuenta</span>
          </button>
        </nav>
      </div>
    </AuthContext.Provider>
  );
}

// ==========================================
// 5. SUBPANTALLAS
// ==========================================

// --- INICIO ---
function PantallaInicio({ irA, totalCreditos, totalFletes }: { irA: (p: any) => void; totalCreditos: number; totalFletes: number }) {
  return (
    <div style={estilos.vista}>
      <div style={estilos.tarjetaAlertaSeguridad}>
        <strong>🛡️ Transparencia CampoAsiste:</strong>
        <p style={{ margin: '4px 0 0 0', fontSize: '13px' }}>
          Esta plataforma <strong>no presta dinero ni cobra comisiones por adelantado</strong>. Conectamos solicitudes con cooperativas y transportadores formales. Rechace llamadas que le exijan giros para desembolsos.
        </p>
      </div>

      <div style={estilos.gridBotonesAccion}>
        <button onClick={() => irA('financiacion')} style={estilos.botonAccionVerde}>
          <span style={{ fontSize: '30px' }}>🌱 💵</span>
          <div>
            <div style={estilos.tituloBotonAccion}>Solicitar Financiación</div>
            <div style={estilos.subtituloBotonAccion}>Abonos, semillas y sostenimiento</div>
          </div>
        </button>

        <button onClick={() => irA('transporte')} style={estilos.botonAccionTierra}>
          <span style={{ fontSize: '30px' }}>🚚 📦</span>
          <div>
            <div style={estilos.tituloBotonAccion}>Buscar Transporte</div>
            <div style={estilos.subtituloBotonAccion}>Fletes directos para su cosecha</div>
          </div>
        </button>
      </div>

      <div style={estilos.filaResumen}>
        <div style={estilos.cajaContador}>
          <span style={estilos.numeroContador}>{totalCreditos}</span>
          <span style={estilos.etiquetaContador}>Solicitudes en curso</span>
        </div>
        <div style={estilos.cajaContador}>
          <span style={estilos.numeroContador}>{totalFletes}</span>
          <span style={estilos.etiquetaContador}>Cargas disponibles</span>
        </div>
      </div>

      <div style={estilos.tarjetaBlanca}>
        <h3 style={estilos.tituloSeccion}>📌 Pasos para pedir financiación con éxito</h3>
        <ol style={{ paddingLeft: '20px', margin: 0, fontSize: '13px', lineHeight: '1.6', color: '#374151' }}>
          <li>Calcule cuántas hectáreas trabajará y cuánto abono requiere.</li>
          <li>Indique la fecha exacta en la que comenzará la preparación del suelo.</li>
          <li>La cooperativa asignada revisará su solicitud y lo contactará.</li>
        </ol>
      </div>
    </div>
  );
}

// --- FINANCIACIÓN ---
function PantallaFinanciacion({
  solicitudes,
  usuario,
  formatoCOP,
  onCrear
}: {
  solicitudes: SolicitudFinanciacion[];
  usuario: Usuario | null;
  formatoCOP: (n: number) => string;
  onCrear: (s: SolicitudFinanciacion) => void;
}) {
  const [abrirFormulario, setAbrirFormulario] = useState(false);
  const [enviando, setEnviando] = useState(false);

  const [cultivo, setCultivo] = useState('');
  const [depto, setDepto] = useState('Boyacá');
  const [municipio, setMunicipio] = useState('Ventaquemada');
  const [monto, setMonto] = useState<number | ''>('');
  const [hectareas, setHectareas] = useState<number | ''>('');
  const [finalidad, setFinalidad] = useState<SolicitudFinanciacion['finalidad']>('Fertilizantes');
  const [fechaRequerida, setFechaRequerida] = useState('');
  const [descripcion, setDescripcion] = useState('');
  const [entidad, setEntidad] = useState('Coop Agrocredit Boyacá (Demostración)');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!usuario) {
      alert('Debe iniciar sesión para solicitar financiación.');
      return;
    }
    if (usuario.rol !== 'agricultor') {
      alert('Solo los usuarios registrados como agricultores pueden solicitar financiación.');
      return;
    }
    if (!cultivo || !monto || Number(monto) <= 0 || !hectareas || !fechaRequerida) {
      alert('Por favor complete todos los campos obligatorios (*).');
      return;
    }

    setEnviando(true);
    const nueva: SolicitudFinanciacion = {
      id: `FIN-${new Date().getFullYear()}-${Math.floor(100 + Math.random() * 900)}`,
      agricultorId: usuario.id,
      agricultorNombre: usuario.nombre,
      cultivo,
      departamento: depto,
      municipio,
      montoCop: Number(monto),
      areaHectareas: Number(hectareas),
      finalidad,
      descripcion,
      fechaRequerida,
      entidadPreferida: entidad,
      estado: 'Pendiente de revisión',
      fechaCreacion: new Date().toISOString().split('T')[0],
      historial: [
        {
          fecha: new Date().toISOString().split('T')[0],
          estado: 'Pendiente de revisión',
          nota: 'Solicitud radicada en la plataforma.',
          autor: usuario.nombre
        }
      ]
    };

    onCrear(nueva);
    setEnviando(false);
    setAbrirFormulario(false);
    setCultivo('');
    setMonto('');
    setHectareas('');
  };

  const misSolicitudes = usuario?.rol === 'agricultor' 
    ? solicitudes.filter(s => s.agricultorId === usuario.id)
    : solicitudes;

  return (
    <div style={estilos.vista}>
      <div style={estilos.encabezadoSeccionRow}>
        <h2 style={estilos.tituloPagina}>Crédito Agrícola</h2>
        {usuario?.rol === 'agricultor' && (
          <button onClick={() => setAbrirFormulario(!abrirFormulario)} style={estilos.botonPrincipalChico}>
            {abrirFormulario ? 'Cancelar' : '+ Solicitar'}
          </button>
        )}
      </div>

      {abrirFormulario && (
        <form onSubmit={handleSubmit} style={estilos.formularioCaja}>
          <h3 style={estilos.tituloFormulario}>Nueva Solicitud de Crédito</h3>

          <label style={estilos.label}>Cultivo principal *</label>
          <input 
            type="text" 
            placeholder="Ej: Papa, Plátano, Café, Maíz" 
            value={cultivo} 
            onChange={(e) => setCultivo(e.target.value)} 
            style={estilos.inputTexto}
          />

          <div style={estilos.dosColumnas}>
            <div>
              <label style={estilos.label}>Departamento *</label>
              <select value={depto} onChange={(e) => { setDepto(e.target.value); setMunicipio(MUNICIPIOS_COLOMBIA[e.target.value][0]); }} style={estilos.inputTexto}>
                {Object.keys(MUNICIPIOS_COLOMBIA).map(d => <option key={d} value={d}>{d}</option>)}
              </select>
            </div>
            <div>
              <label style={estilos.label}>Municipio *</label>
              <select value={municipio} onChange={(e) => setMunicipio(e.target.value)} style={estilos.inputTexto}>
                {MUNICIPIOS_COLOMBIA[depto]?.map(m => <option key={m} value={m}>{m}</option>)}
              </select>
            </div>
          </div>

          <div style={estilos.dosColumnas}>
            <div>
              <label style={estilos.label}>Monto requerido (COP) *</label>
              <input 
                type="number" 
                placeholder="Ej: 5000000" 
                value={monto} 
                onChange={(e) => setMonto(Number(e.target.value))} 
                style={estilos.inputTexto}
              />
            </div>
            <div>
              <label style={estilos.label}>Hectáreas sembradas *</label>
              <input 
                type="number" 
                step="0.1" 
                placeholder="Ej: 1.5" 
                value={hectareas} 
                onChange={(e) => setHectareas(Number(e.target.value))} 
                style={estilos.inputTexto}
              />
            </div>
          </div>

          <label style={estilos.label}>Destino de los fondos *</label>
          <select value={finalidad} onChange={(e) => setFinalidad(e.target.value as any)} style={estilos.inputTexto}>
            <option value="Fertilizantes">Abonos y Fertilizantes</option>
            <option value="Semillas">Semilla certificada</option>
            <option value="Maquinaria">Alquiler o arreglo de maquinaria</option>
            <option value="Adecuación de tierra">Preparación del terreno y zanjas</option>
            <option value="Insumos varios">Insumos y sanidad</option>
          </select>

          <label style={estilos.label}>¿Para cuándo necesita los recursos? *</label>
          <input type="date" value={fechaRequerida} onChange={(e) => setFechaRequerida(e.target.value)} style={estilos.inputTexto} />

          <label style={estilos.label}>Entidad financiera asignada</label>
          <select value={entidad} onChange={(e) => setEntidad(e.target.value)} style={estilos.inputTexto}>
            <option value="Coop Agrocredit Boyacá (Demostración)">Coop Agrocredit Boyacá (Demostración)</option>
            <option value="Banco Agrícola Comunitario (Demostración)">Banco Agrícola Comunitario (Demostración)</option>
          </select>

          <label style={estilos.label}>Descripción de la finca o proyecto (opcional)</label>
          <textarea rows={2} placeholder="Comente si cuenta con agua de reservorio o acueducto..." value={descripcion} onChange={(e) => setDescripcion(e.target.value)} style={estilos.textarea} />

          <button type="submit" disabled={enviando} style={enviando ? estilos.botonInactivo : estilos.botonEnviarFormulario}>
            {enviando ? 'Guardando...' : 'Radicar Solicitud Oficial'}
          </button>
        </form>
      )}

      {/* Lista de solicitudes */}
      <h3 style={estilos.subtituloLista}>
        {usuario?.rol === 'agricultor' ? 'Mis Solicitudes Radicadas' : 'Todas las solicitudes'}
      </h3>

      {misSolicitudes.length === 0 ? (
        <div style={estilos.tarjetaVacia}>No registra solicitudes de crédito activas.</div>
      ) : (
        misSolicitudes.map(sol => (
          <div key={sol.id} style={estilos.tarjetaItem}>
            <div style={estilos.tarjetaHeader}>
              <span style={estilos.badgeId}>{sol.id}</span>
              <span style={estilosBadgeEstado(sol.estado)}>{sol.estado}</span>
            </div>
            <h4 style={{ margin: '4px 0', fontSize: '16px', color: '#1B4332' }}>{sol.cultivo}</h4>
            <div style={estilos.lineaDatos}>
              <span>📍 {sol.municipio}, {sol.departamento}</span>
              <span>📐 {sol.areaHectareas} ha</span>
            </div>
            <div style={{ margin: '6px 0', fontSize: '15px' }}>
              Monto: <strong>{formatoCOP(sol.montoCop)}</strong> para {sol.finalidad}
            </div>
            <div style={estilos.cajaHistorial}>
              <span style={{ fontSize: '11px', color: '#6B7280', fontWeight: 'bold' }}>Último movimiento:</span>
              <p style={{ margin: 0, fontSize: '12px' }}>
                {sol.historial[sol.historial.length - 1]?.fecha}: {sol.historial[sol.historial.length - 1]?.nota}
              </p>
            </div>
          </div>
        ))
      )}
    </div>
  );
}

// --- TRANSPORTE ---
function PantallaTransporte({
  transportes,
  usuario,
  formatoCOP,
  onCrearCarga,
  onEnviarPropuesta,
  onAceptarPropuesta
}: {
  transportes: PublicacionTransporte[];
  usuario: Usuario | null;
  formatoCOP: (n: number) => string;
  onCrearCarga: (c: PublicacionTransporte) => void;
  onEnviarPropuesta: (p: PropuestaFlete) => void;
  onAceptarPropuesta: (id: string, p: PropuestaFlete) => void;
}) {
  const [abrirFormulario, setAbrirFormulario] = useState(false);
  const [propuestaModalCarga, setPropuestaModalCarga] = useState<PublicacionTransporte | null>(null);

  // Formulario de carga
  const [producto, setProducto] = useState('');
  const [cantidad, setCantidad] = useState<number | ''>('');
  const [unidad, setUnidad] = useState<PublicacionTransporte['unidad']>('bultos');
  const [origenDepto, setOrigenDepto] = useState('Boyacá');
  const [origenMuni, setOrigenMuni] = useState('Ventaquemada');
  const [vereda, setVereda] = useState('');
  const [destinoDepto, setDestinoDepto] = useState('Cundinamarca');
  const [destinoMuni, setDestinoMuni] = useState('Bogotá D.C.');
  const [fechaRecogida, setFechaRecogida] = useState('');
  const [vehiculo, setVehiculo] = useState('Camión Turbo NPR');
  const [pago, setPago] = useState<number | ''>('');

  // Formulario de propuesta (transportador)
  const [precioOferta, setPrecioOferta] = useState<number | ''>('');
  const [vehiculoOferta, setVehiculoOferta] = useState('');
  const [placaOferta, setPlacaOferta] = useState('');
  const [mensajeOferta, setMensajeOferta] = useState('');

  const guardarCarga = (e: React.FormEvent) => {
    e.preventDefault();
    if (!usuario) {
      alert('Debe iniciar sesión para publicar.');
      return;
    }
    if (!producto || !cantidad || !fechaRecogida) {
      alert('Complete los datos obligatorios del viaje.');
      return;
    }

    const nueva: PublicacionTransporte = {
      id: `TRP-${new Date().getFullYear()}-${Math.floor(100 + Math.random() * 900)}`,
      agricultorId: usuario.id,
      agricultorNombre: usuario.nombre,
      producto,
      cantidadValor: Number(cantidad),
      unidad,
      origenDepto,
      origenMunicipio: origenMuni,
      veredaDetallada: vereda || 'Sector rural principal',
      destinoDepto,
      destinoMunicipio: destinoMuni,
      fechaRecogida,
      tipoVehiculoSugerido: vehiculo,
      pagoOfrecidoCop: pago ? Number(pago) : 'Negociable',
      estado: 'Abierta',
      telefonoContacto: usuario.telefono,
      propuestas: []
    };

    onCrearCarga(nueva);
    setAbrirFormulario(false);
  };

  const enviarPropuestaFlete = (e: React.FormEvent) => {
    e.preventDefault();
    if (!usuario || !propuestaModalCarga) return;
    if (!precioOferta || !placaOferta) {
      alert('Indique el precio y la placa de su vehículo.');
      return;
    }

    const prop: PropuestaFlete = {
      id: `PROP-${Date.now()}`,
      transporteId: propuestaModalCarga.id,
      transportadorId: usuario.id,
      transportadorNombre: usuario.nombre,
      telefono: usuario.telefono,
      vehiculo: vehiculoOferta || 'Camión disponible',
      placa: placaOferta.toUpperCase(),
      precioCop: Number(precioOferta),
      mensaje: mensajeOferta,
      fecha: new Date().toISOString().split('T')[0]
    };

    onEnviarPropuesta(prop);
    setPropuestaModalCarga(null);
    setPrecioOferta('');
    setPlacaOferta('');
    setMensajeOferta('');
  };

  return (
    <div style={estilos.vista}>
      <div style={estilos.encabezadoSeccionRow}>
        <h2 style={estilos.tituloPagina}>Transporte de Cosechas</h2>
        {usuario?.rol === 'agricultor' && (
          <button onClick={() => setAbrirFormulario(!abrirFormulario)} style={estilos.botonPrincipalChico}>
            {abrirFormulario ? 'Cancelar' : '+ Publicar Carga'}
          </button>
        )}
      </div>

      {abrirFormulario && (
        <form onSubmit={guardarCarga} style={estilos.formularioCaja}>
          <h3 style={estilos.tituloFormulario}>Publicar Viaje de Cosecha</h3>

          <label style={estilos.label}>Producto *</label>
          <input type="text" placeholder="Ej: Papa, Cebolla junca, Café" value={producto} onChange={(e) => setProducto(e.target.value)} style={estilos.inputTexto} />

          <div style={estilos.dosColumnas}>
            <div>
              <label style={estilos.label}>Cantidad *</label>
              <input type="number" placeholder="Ej: 100" value={cantidad} onChange={(e) => setCantidad(Number(e.target.value))} style={estilos.inputTexto} />
            </div>
            <div>
              <label style={estilos.label}>Unidad *</label>
              <select value={unidad} onChange={(e) => setUnidad(e.target.value as any)} style={estilos.inputTexto}>
                <option value="bultos">Bultos</option>
                <option value="cargas">Cargas</option>
                <option value="toneladas">Toneladas</option>
                <option value="kg">Kilogramos</option>
              </select>
            </div>
          </div>

          <label style={estilos.label}>Municipio de recogida (Origen) *</label>
          <select value={origenMuni} onChange={(e) => setOrigenMuni(e.target.value)} style={estilos.inputTexto}>
            {MUNICIPIOS_COLOMBIA['Boyacá'].map(m => <option key={m} value={m}>{m} (Boyacá)</option>)}
          </select>

          <label style={estilos.label}>Vereda o sector (Privado hasta acordar)</label>
          <input type="text" placeholder="Ej: Vereda Montoya, entrada por la escuela" value={vereda} onChange={(e) => setVereda(e.target.value)} style={estilos.inputTexto} />

          <label style={estilos.label}>Municipio de entrega (Destino) *</label>
          <input type="text" placeholder="Ej: Corabastos Bogotá, Paloquemao, Plaza Duitama" value={destinoMuni} onChange={(e) => setDestinoMuni(e.target.value)} style={estilos.inputTexto} />

          <div style={estilos.dosColumnas}>
            <div>
              <label style={estilos.label}>Fecha de recogida *</label>
              <input type="date" value={fechaRecogida} onChange={(e) => setFechaRecogida(e.target.value)} style={estilos.inputTexto} />
            </div>
            <div>
              <label style={estilos.label}>Vehículo requerido</label>
              <select value={vehiculo} onChange={(e) => setVehiculo(e.target.value)} style={estilos.inputTexto}>
                <option value="Camión Turbo NPR">Turbo NPR (hasta 4.5 ton)</option>
                <option value="Camioneta Estacas">Camioneta (hasta 1.5 ton)</option>
                <option value="Camión Sencillo">Sencillo (hasta 8 ton)</option>
              </select>
            </div>
          </div>

          <label style={estilos.label}>Flete ofrecido (COP, dejar vacío si es a convenir)</label>
          <input type="number" placeholder="Ej: 1400000" value={pago} onChange={(e) => setPago(Number(e.target.value))} style={estilos.inputTexto} />

          <button type="submit" style={estilos.botonEnviarFormulario}>Publicar para los transportadores</button>
        </form>
      )}

      {/* Lista de cargas */}
      <div style={{ display: 'flex', flexDirection: 'column', gap: '12px' }}>
        {transportes.map(t => {
          const fleteAcordado = t.estado === 'Transporte acordado';
          const soyElPublicador = usuario?.id === t.agricultorId;
          const soyElTransportadorElegido = usuario?.id === t.transportadorAcordadoId;
          const puedeVerContacto = soyElPublicador || soyElTransportadorElegido || usuario?.rol === 'admin';

          return (
            <div key={t.id} style={estilos.tarjetaItem}>
              <div style={estilos.tarjetaHeader}>
                <strong style={{ fontSize: '15px', color: '#1B4332' }}>{t.producto}</strong>
                <span style={{ backgroundColor: '#FEF3C7', color: '#92400E', padding: '2px 8px', borderRadius: '4px', fontSize: '11px', fontWeight: 'bold' }}>
                  {t.estado}
                </span>
              </div>

              <p style={{ margin: '4px 0', fontSize: '13px', color: '#374151' }}>
                📍 <strong>Origen:</strong> {t.origenMunicipio} ({t.origenDepto}) <br />
                🏁 <strong>Destino:</strong> {t.destinoMunicipio}
              </p>

              <div style={estilos.lineaDatos}>
                <span>📦 {t.cantidadValor} {t.unidad}</span>
                <span>📅 {t.fechaRecogida}</span>
                <span>🚛 {t.tipoVehiculoSugerido}</span>
              </div>

              <div style={{ margin: '6px 0', fontSize: '14px' }}>
                Valor ofrecido: <strong>{typeof t.pagoOfrecidoCop === 'number' ? formatoCOP(t.pagoOfrecidoCop) : 'A convenir'}</strong>
              </div>

              {/* Protección de datos personales */}
              <div style={{ backgroundColor: '#F3F4F6', padding: '8px', borderRadius: '6px', fontSize: '12px', margin: '6px 0' }}>
                {puedeVerContacto ? (
                  <div>
                    🔒 <strong>Datos de contacto habilitados:</strong><br />
                    Agricultor: {t.agricultorNombre} - Tel: <strong>{t.telefonoContacto}</strong><br />
                    Vereda: {t.veredaDetallada}
                  </div>
                ) : (
                  <span style={{ color: '#6B7280' }}>
                    🔒 El teléfono y la vereda se reservan hasta que ambas partes acuerden el flete.
                  </span>
                )}
              </div>

              {/* Botón para enviar propuesta */}
              {usuario?.rol === 'transportador' && !fleteAcordado && (
                <button onClick={() => setPropuestaModalCarga(t)} style={estilos.botonPropuesta}>
                  ✍️ Enviar Propuesta de Flete
                </button>
              )}

              {/* Lista de propuestas recibidas (visible para el agricultor dueño) */}
              {soyElPublicador && t.propuestas.length > 0 && (
                <div style={{ marginTop: '8px', borderTop: '1px solid #E5E7EB', paddingTop: '8px' }}>
                  <span style={{ fontSize: '12px', fontWeight: 'bold', color: '#1B4332' }}>Propuestas recibidas ({t.propuestas.length}):</span>
                  {t.propuestas.map(p => (
                    <div key={p.id} style={{ backgroundColor: '#F0FDF4', padding: '8px', borderRadius: '6px', margin: '6px 0', border: '1px solid #BBF7D0' }}>
                      <div style={{ display: 'flex', justifyContent: 'space-between', fontSize: '13px' }}>
                        <strong>{p.transportadorNombre}</strong>
                        <span style={{ color: '#1B4332', fontWeight: 'bold' }}>{formatoCOP(p.precioCop)}</span>
                      </div>
                      <div style={{ fontSize: '12px', color: '#4B5563' }}>
                        Camión: {p.vehiculo} (Placa: {p.placa}) <br />
                        Nota: "{p.mensaje}"
                      </div>
                      {!fleteAcordado && (
                        <button onClick={() => onAceptarPropuesta(t.id, p)} style={estilos.botonAceptarFlete}>
                          🤝 Aceptar y Compartir Teléfono
                        </button>
                      )}
                    </div>
                  ))}
                </div>
              )}
            </div>
          );
        })}
      </div>

      {/* Modal para que el transportador envíe propuesta */}
      {propuestaModalCarga && (
        <div style={estilos.modalOverlay}>
          <div style={estilos.modalCaja}>
            <h3 style={estilos.tituloFormulario}>Enviar Oferta de Transporte</h3>
            <p style={{ margin: 0, fontSize: '13px', color: '#666' }}>
              Carga: {propuestaModalCarga.producto} ({propuestaModalCarga.origenMunicipio} ➔ {propuestaModalCarga.destinoMunicipio})
            </p>

            <form onSubmit={enviarPropuestaFlete} style={{ display: 'flex', flexDirection: 'column', gap: '8px', marginTop: '10px' }}>
              <label style={estilos.label}>Precio del viaje (COP) *</label>
              <input type="number" placeholder="Ej: 1400000" value={precioOferta} onChange={(e) => setPrecioOferta(Number(e.target.value))} style={estilos.inputTexto} />

              <label style={estilos.label}>Placa del vehículo *</label>
              <input type="text" placeholder="Ej: XYZ-123" value={placaOferta} onChange={(e) => setPlacaOferta(e.target.value)} style={estilos.inputTexto} />

              <label style={estilos.label}>Modelo y tipo de carrocería</label>
              <input type="text" placeholder="Ej: Turbo NPR con carpa estanca" value={vehiculoOferta} onChange={(e) => setVehiculoOferta(e.target.value)} style={estilos.inputTexto} />

              <label style={estilos.label}>Mensaje para el agricultor</label>
              <textarea rows={2} placeholder="Ej: Llego a primera hora con dos ayudantes..." value={mensajeOferta} onChange={(e) => setMensajeOferta(e.target.value)} style={estilos.textarea} />

              <div style={estilos.dosColumnas}>
                <button type="button" onClick={() => setPropuestaModalCarga(null)} style={estilos.botonSecundario}>Cancelar</button>
                <button type="submit" style={estilos.botonEnviarFormulario}>Enviar Oferta</button>
              </div>
            </form>
          </div>
        </div>
      )}
    </div>
  );
}

// --- BANCO Y COOPERATIVAS ---
function PantallaBanco({
  solicitudes,
  usuario,
  formatoCOP,
  onActualizar
}: {
  solicitudes: SolicitudFinanciacion[];
  usuario: Usuario | null;
  formatoCOP: (n: number) => string;
  onActualizar: (id: string, nuevoEstado: EstadoFinanciacion, nota: string) => void;
}) {
  const [seleccionada, setSeleccionada] = useState<SolicitudFinanciacion | null>(null);
  const [nuevoEstado, setNuevoEstado] = useState<EstadoFinanciacion>('En evaluación');
  const [nota, setNota] = useState('');

  if (usuario?.rol !== 'entidad' && usuario?.rol !== 'admin') {
    return (
      <div style={estilos.vista}>
        <div style={estilos.tarjetaAlertaSeguridad}>
          Acceso restringido: esta sección está reservada para analistas de entidades financieras autorizadas.
        </div>
      </div>
    );
  }

  return (
    <div style={estilos.vista}>
      <h2 style={estilos.tituloPagina}>Portal de Entidades Financieras</h2>
      <p style={{ margin: '0 0 12px 0', fontSize: '13px', color: '#6B7280' }}>
        Estudio de capacidad productiva y viabilidad de crédito agropecuario.
      </p>

      <div style={{ display: 'flex', flexDirection: 'column', gap: '10px' }}>
        {solicitudes.map(sol => (
          <div key={sol.id} style={estilos.tarjetaItem}>
            <div style={estilos.tarjetaHeader}>
              <strong>{sol.id} - {sol.cultivo}</strong>
              <span style={estilosBadgeEstado(sol.estado)}>{sol.estado}</span>
            </div>
            <p style={{ margin: '4px 0', fontSize: '13px' }}>
              Agricultor: <strong>{sol.agricultorNombre}</strong> | {sol.municipio}, {sol.departamento}
            </p>
            <p style={{ margin: '4px 0', fontSize: '14px' }}>
              Monto: <strong>{formatoCOP(sol.montoCop)}</strong> ({sol.areaHectareas} hectáreas)
            </p>

            <button onClick={() => { setSeleccionada(sol); setNuevoEstado(sol.estado); }} style={estilos.botonDictamen}>
              ⚖️ Evaluar y Actualizar Estado
            </button>
          </div>
        ))}
      </div>

      {seleccionada && (
        <div style={estilos.modalOverlay}>
          <div style={estilos.modalCaja}>
            <h3 style={estilos.tituloFormulario}>Decisión sobre Solicitud #{seleccionada.id}</h3>
            <p style={{ fontSize: '13px', margin: '4px 0' }}>Cultivo: {seleccionada.cultivo} - Solicitante: {seleccionada.agricultorNombre}</p>

            <label style={estilos.label}>Nuevo Estado:</label>
            <select value={nuevoEstado} onChange={(e) => setNuevoEstado(e.target.value as any)} style={estilos.inputTexto}>
              <option value="En evaluación">En evaluación</option>
              <option value="Requiere información adicional">Requiere información adicional (Fiador / Papeles)</option>
              <option value="Aprobada por la entidad">Aprobada por la entidad</option>
              <option value="Rechazada por la entidad">Rechazada por la entidad</option>
            </select>

            <label style={estilos.label}>Nota oficial para el agricultor:</label>
            <textarea rows={3} placeholder="Explique las condiciones o la razón técnica..." value={nota} onChange={(e) => setNota(e.target.value)} style={estilos.textarea} />

            <div style={estilos.dosColumnas}>
              <button onClick={() => setSeleccionada(null)} style={estilos.botonSecundario}>Cerrar</button>
              <button onClick={() => { onActualizar(seleccionada.id, nuevoEstado, nota); setSeleccionada(null); setNota(''); }} style={estilos.botonEnviarFormulario}>
                Guardar Dictamen
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}

// --- ADMINISTRADOR ---
function PantallaAdmin({
  solicitudes,
  transportes,
  usuarios,
  usuario,
  onEliminarTransporte
}: {
  solicitudes: SolicitudFinanciacion[];
  transportes: PublicacionTransporte[];
  usuarios: Usuario[];
  usuario: Usuario | null;
  onEliminarTransporte: (id: string) => void;
}) {
  if (usuario?.rol !== 'admin') {
    return (
      <div style={estilos.vista}>
        <div style={estilos.tarjetaAlertaSeguridad}>Área exclusiva de administración del sistema.</div>
      </div>
    );
  }

  return (
    <div style={estilos.vista}>
      <h2 style={estilos.tituloPagina}>Panel de Administración y Moderación</h2>

      <div style={estilos.filaResumen}>
        <div style={estilos.cajaContador}>
          <span style={estilos.numeroContador}>{usuarios.length}</span>
          <span style={estilos.etiquetaContador}>Usuarios</span>
        </div>
        <div style={estilos.cajaContador}>
          <span style={estilos.numeroContador}>{solicitudes.length}</span>
          <span style={estilos.etiquetaContador}>Créditos</span>
        </div>
        <div style={estilos.cajaContador}>
          <span style={estilos.numeroContador}>{transportes.length}</span>
          <span style={estilos.etiquetaContador}>Fletes</span>
        </div>
      </div>

      <h3 style={estilos.subtituloLista}>Moderación de Viajes de Transporte</h3>
      {transportes.map(t => (
        <div key={t.id} style={estilos.tarjetaItem}>
          <div style={estilos.tarjetaHeader}>
            <span>{t.id} - {t.producto}</span>
            <button onClick={() => onEliminarTransporte(t.id)} style={{ backgroundColor: '#FEE2E2', color: '#991B1B', border: 'none', padding: '4px 8px', borderRadius: '4px', fontSize: '11px', cursor: 'pointer' }}>
              Eliminar
            </button>
          </div>
          <p style={{ margin: '4px 0', fontSize: '12px' }}>Publicado por: {t.agricultorNombre} ({t.telefonoContacto})</p>
        </div>
      ))}
    </div>
  );
}

// --- PERFIL Y AUTENTICACIÓN ---
function PantallaPerfil({
  usuario,
  usuariosDemo,
  onCambiarSesionRapida
}: {
  usuario: Usuario | null;
  usuariosDemo: Usuario[];
  onCambiarSesionRapida: (correo: string) => void;
}) {
  const { registrarse, iniciarSesion } = useAuth();
  const [modoRegistro, setModoRegistro] = useState(false);

  // Registro state
  const [nombre, setNombre] = useState('');
  const [correo, setCorreo] = useState('');
  const [telefono, setTelefono] = useState('');
  const [rol, setRol] = useState<RolUsuario>('agricultor');
  const [depto, setDepto] = useState('Boyacá');
  const [municipio, setMunicipio] = useState('Ventaquemada');

  const ejecutarRegistro = (e: React.FormEvent) => {
    e.preventDefault();
    if (!nombre || !correo || !telefono) {
      alert('Por favor complete todos los datos.');
      return;
    }
    registrarse({ nombre, correo, telefono, rol, departamento: depto, municipio });
    setModoRegistro(false);
  };

  return (
    <div style={estilos.vista}>
      <h2 style={estilos.tituloPagina}>Mi Perfil</h2>

      {usuario ? (
        <div>
          <div style={estilos.tarjetaPerfil}>
            <div style={estilos.avatar}>👨‍🌾</div>
            <div>
              <h3 style={{ margin: 0 }}>{usuario.nombre}</h3>
              <p style={{ margin: '2px 0', color: '#2D6A4F', fontWeight: 'bold' }}>Rol: {usuario.rol.toUpperCase()}</p>
              <p style={{ margin: 0, fontSize: '13px', color: '#666' }}>📍 {usuario.municipio}, {usuario.departamento}</p>
              <p style={{ margin: 0, fontSize: '13px', color: '#666' }}>📞 {usuario.telefono}</p>
            </div>
          </div>

          <div style={estilos.tarjetaBlanca}>
            <h4 style={{ margin: '0 0 6px 0', fontSize: '14px', color: '#1B4332' }}>🛡️ Habeas Data y Privacidad (Ley 1581 de 2012)</h4>
            <p style={{ fontSize: '12px', color: '#4B5563', lineHeight: '1.4', margin: 0 }}>
              Sus datos personales están amparados por la legislación colombiana. CampoAsiste no expone su número celular ni la localización de sus predios a usuarios desconocidos.
            </p>
          </div>
        </div>
      ) : (
        <div>
          {!modoRegistro ? (
            <div style={estilos.formularioCaja}>
              <h3 style={estilos.tituloFormulario}>Iniciar Sesión</h3>
              <p style={{ fontSize: '12px', color: '#666', margin: 0 }}>Seleccione una cuenta de prueba rápida:</p>

              <div style={{ display: 'flex', flexDirection: 'column', gap: '6px', margin: '10px 0' }}>
                {usuariosDemo.map(u => (
                  <button key={u.id} onClick={() => onCambiarSesionRapida(u.correo)} style={estilos.botonCuentaRapida}>
                    {u.nombre} <em>({u.rol})</em>
                  </button>
                ))}
              </div>

              <button onClick={() => setModoRegistro(true)} style={estilos.botonSecundario}>
                Crear una cuenta nueva
              </button>
            </div>
          ) : (
            <form onSubmit={ejecutarRegistro} style={estilos.formularioCaja}>
              <h3 style={estilos.tituloFormulario}>Registro de Usuario</h3>

              <label style={estilos.label}>Nombre completo *</label>
              <input type="text" placeholder="Ej: Pedro Nel Gómez" value={nombre} onChange={(e) => setNombre(e.target.value)} style={estilos.inputTexto} />

              <label style={estilos.label}>Correo electrónico *</label>
              <input type="email" placeholder="correo@ejemplo.com" value={correo} onChange={(e) => setCorreo(e.target.value)} style={estilos.inputTexto} />

              <label style={estilos.label}>Teléfono celular *</label>
              <input type="tel" placeholder="Ej: 312 345 6789" value={telefono} onChange={(e) => setTelefono(e.target.value)} style={estilos.inputTexto} />

              <label style={estilos.label}>Tipo de usuario (Rol) *</label>
              <select value={rol} onChange={(e) => setRol(e.target.value as any)} style={estilos.inputTexto}>
                <option value="agricultor">🌾 Agricultor</option>
                <option value="transportador">🚛 Transportador</option>
                <option value="entidad">🏦 Cooperativa / Banco</option>
              </select>

              <div style={estilos.dosColumnas}>
                <button type="button" onClick={() => setModoRegistro(false)} style={estilos.botonSecundario}>Volver</button>
                <button type="submit" style={estilos.botonEnviarFormulario}>Crear Cuenta</button>
              </div>
            </form>
          )}
        </div>
      )}
    </div>
  );
}

// ==========================================
// 6. ESTILOS Y HELPERS
// ==========================================

function estilosBadgeEstado(estado: EstadoFinanciacion): React.CSSProperties {
  switch (estado) {
    case 'Aprobada por la entidad':
      return { backgroundColor: '#D8F3DC', color: '#1B4332', padding: '3px 8px', borderRadius: '4px', fontSize: '11px', fontWeight: 'bold' };
    case 'Rechazada por la entidad':
      return { backgroundColor: '#FEE2E2', color: '#991B1B', padding: '3px 8px', borderRadius: '4px', fontSize: '11px', fontWeight: 'bold' };
    case 'Requiere información adicional':
      return { backgroundColor: '#FEF3C7', color: '#92400E', padding: '3px 8px', borderRadius: '4px', fontSize: '11px', fontWeight: 'bold' };
    default:
      return { backgroundColor: '#DBEAFE', color: '#1E40AF', padding: '3px 8px', borderRadius: '4px', fontSize: '11px', fontWeight: 'bold' };
  }
}

const estilos: Record<string, React.CSSProperties> = {
  contenedorApp: {
    maxWidth: '520px',
    margin: '0 auto',
    minHeight: '100vh',
    backgroundColor: '#F9FAFB',
    fontFamily: 'system-ui, -apple-system, Roboto, sans-serif',
    display: 'flex',
    flexDirection: 'column',
    position: 'relative',
    paddingBottom: '76px',
    boxShadow: '0 0 15px rgba(0,0,0,0.05)'
  },
  bannerSinRed: {
    backgroundColor: '#C57B57',
    color: '#FFF',
    padding: '8px 12px',
    fontSize: '12px',
    textAlign: 'center',
    fontWeight: 'bold'
  },
  alertaFlotante: {
    position: 'fixed',
    top: '12px',
    left: '50%',
    transform: 'translateX(-50%)',
    color: '#FFF',
    padding: '10px 18px',
    borderRadius: '8px',
    zIndex: 9999,
    fontSize: '13px',
    fontWeight: 'bold',
    maxWidth: '85%',
    textAlign: 'center',
    boxShadow: '0 4px 10px rgba(0,0,0,0.15)'
  },
  cabecera: {
    backgroundColor: '#1B4332',
    color: '#FFF',
    padding: '12px 16px',
    display: 'flex',
    flexDirection: 'column',
    gap: '8px'
  },
  cabeceraSuperior: {
    display: 'flex',
    alignItems: 'center',
    gap: '10px'
  },
  logoIcono: {
    fontSize: '26px',
    backgroundColor: '#2D6A4F',
    borderRadius: '10px',
    padding: '6px'
  },
  tituloLogo: {
    margin: 0,
    fontSize: '20px',
    fontWeight: 'bold'
  },
  eslogan: {
    margin: 0,
    fontSize: '11px',
    color: '#B7E4C7'
  },
  barraUsuario: {
    borderTop: '1px solid rgba(255,255,255,0.15)',
    paddingTop: '6px'
  },
  usuarioInfoRow: {
    display: 'flex',
    justifyContent: 'space-between',
    alignItems: 'center',
    fontSize: '12px'
  },
  botonCerrarSesion: {
    backgroundColor: '#2D6A4F',
    color: '#FFF',
    border: 'none',
    padding: '4px 8px',
    borderRadius: '4px',
    fontSize: '11px',
    cursor: 'pointer'
  },
  textoInvitado: {
    fontSize: '11px',
    color: '#D8F3DC'
  },
  cuerpoPrincipal: {
    flex: 1,
    padding: '14px'
  },
  vista: {
    display: 'flex',
    flexDirection: 'column',
    gap: '14px'
  },
  tarjetaAlertaSeguridad: {
    backgroundColor: '#FEF3C7',
    borderLeft: '4px solid #D97706',
    padding: '10px',
    borderRadius: '6px',
    color: '#92400E',
    fontSize: '13px'
  },
  gridBotonesAccion: {
    display: 'flex',
    flexDirection: 'column',
    gap: '10px'
  },
  botonAccionVerde: {
    backgroundColor: '#2D6A4F',
    color: '#FFF',
    border: 'none',
    borderRadius: '12px',
    padding: '16px',
    display: 'flex',
    alignItems: 'center',
    gap: '12px',
    cursor: 'pointer',
    textAlign: 'left'
  },
  botonAccionTierra: {
    backgroundColor: '#D4A373',
    color: '#1F2937',
    border: 'none',
    borderRadius: '12px',
    padding: '16px',
    display: 'flex',
    alignItems: 'center',
    gap: '12px',
    cursor: 'pointer',
    textAlign: 'left'
  },
  tituloBotonAccion: {
    fontSize: '16px',
    fontWeight: 'bold'
  },
  subtituloBotonAccion: {
    fontSize: '12px',
    opacity: 0.9
  },
  filaResumen: {
    display: 'flex',
    gap: '10px'
  },
  cajaContador: {
    flex: 1,
    backgroundColor: '#FFF',
    padding: '12px',
    borderRadius: '8px',
    border: '1px solid #E5E7EB',
    textAlign: 'center'
  },
  numeroContador: {
    display: 'block',
    fontSize: '22px',
    fontWeight: 'bold',
    color: '#1B4332'
  },
  etiquetaContador: {
    fontSize: '11px',
    color: '#6B7280'
  },
  tarjetaBlanca: {
    backgroundColor: '#FFF',
    padding: '14px',
    borderRadius: '8px',
    border: '1px solid #E5E7EB'
  },
  tituloSeccion: {
    margin: '0 0 10px 0',
    fontSize: '15px',
    color: '#1B4332'
  },
  barraInferior: {
    position: 'fixed',
    bottom: 0,
    left: '50%',
    transform: 'translateX(-50%)',
    width: '100%',
    maxWidth: '520px',
    backgroundColor: '#FFF',
    borderTop: '1px solid #E5E7EB',
    display: 'flex',
    justifyContent: 'space-around',
    padding: '6px 0',
    zIndex: 100
  },
  itemNav: {
    background: 'none',
    border: 'none',
    display: 'flex',
    flexDirection: 'column',
    alignItems: 'center',
    gap: '2px',
    padding: '6px',
    minWidth: '55px',
    cursor: 'pointer'
  },
  iconoNav: {
    fontSize: '18px'
  },
  textoNav: {
    fontSize: '11px',
    fontWeight: 'bold'
  },
  encabezadoSeccionRow: {
    display: 'flex',
    justifyContent: 'space-between',
    alignItems: 'center'
  },
  tituloPagina: {
    margin: 0,
    fontSize: '18px',
    color: '#1B4332'
  },
  botonPrincipalChico: {
    backgroundColor: '#2D6A4F',
    color: '#FFF',
    border: 'none',
    borderRadius: '6px',
    padding: '8px 12px',
    fontSize: '12px',
    fontWeight: 'bold',
    cursor: 'pointer'
  },
  formularioCaja: {
    backgroundColor: '#FFF',
    padding: '14px',
    borderRadius: '10px',
    border: '1px solid #D1D5DB',
    display: 'flex',
    flexDirection: 'column',
    gap: '8px'
  },
  tituloFormulario: {
    margin: '0 0 6px 0',
    fontSize: '15px',
    color: '#1B4332'
  },
  label: {
    fontSize: '12px',
    fontWeight: 'bold',
    color: '#374151'
  },
  inputTexto: {
    padding: '10px',
    borderRadius: '6px',
    border: '1px solid #D1D5DB',
    fontSize: '14px',
    width: '100%',
    boxSizing: 'border-box'
  },
  textarea: {
    padding: '10px',
    borderRadius: '6px',
    border: '1px solid #D1D5DB',
    fontSize: '13px',
    width: '100%',
    boxSizing: 'border-box',
    fontFamily: 'inherit'
  },
  dosColumnas: {
    display: 'grid',
    gridTemplateColumns: '1fr 1fr',
    gap: '8px'
  },
  botonEnviarFormulario: {
    backgroundColor: '#1B4332',
    color: '#FFF',
    padding: '12px',
    borderRadius: '8px',
    border: 'none',
    fontSize: '14px',
    fontWeight: 'bold',
    cursor: 'pointer',
    marginTop: '6px'
  },
  botonInactivo: {
    backgroundColor: '#9CA3AF',
    color: '#FFF',
    padding: '12px',
    borderRadius: '8px',
    border: 'none',
    fontSize: '14px'
  },
  botonSecundario: {
    backgroundColor: '#E5E7EB',
    color: '#374151',
    border: 'none',
    padding: '10px',
    borderRadius: '6px',
    fontSize: '13px',
    cursor: 'pointer'
  },
  subtituloLista: {
    margin: '10px 0 6px 0',
    fontSize: '14px',
    color: '#374151'
  },
  tarjetaVacia: {
    backgroundColor: '#FFF',
    padding: '20px',
    textAlign: 'center',
    borderRadius: '8px',
    color: '#6B7280',
    fontSize: '13px'
  },
  tarjetaItem: {
    backgroundColor: '#FFF',
    padding: '12px',
    borderRadius: '8px',
    border: '1px solid #E5E7EB'
  },
  tarjetaHeader: {
    display: 'flex',
    justifyContent: 'space-between',
    alignItems: 'center'
  },
  badgeId: {
    fontSize: '11px',
    backgroundColor: '#F3F4F6',
    padding: '2px 6px',
    borderRadius: '4px',
    color: '#4B5563',
    fontWeight: 'bold'
  },
  lineaDatos: {
    display: 'flex',
    justifyContent: 'space-between',
    fontSize: '12px',
    color: '#6B7280',
    margin: '4px 0'
  },
  cajaHistorial: {
    backgroundColor: '#F9FAFB',
    padding: '6px 8px',
    borderRadius: '6px',
    marginTop: '6px'
  },
  botonPropuesta: {
    width: '100%',
    backgroundColor: '#D4A373',
    color: '#000',
    border: 'none',
    padding: '10px',
    borderRadius: '6px',
    fontWeight: 'bold',
    fontSize: '13px',
    cursor: 'pointer',
    marginTop: '6px'
  },
  botonAceptarFlete: {
    backgroundColor: '#1B4332',
    color: '#FFF',
    border: 'none',
    padding: '6px 10px',
    borderRadius: '6px',
    fontSize: '12px',
    fontWeight: 'bold',
    cursor: 'pointer',
    marginTop: '6px'
  },
  botonDictamen: {
    width: '100%',
    backgroundColor: '#1B4332',
    color: '#FFF',
    border: 'none',
    padding: '8px',
    borderRadius: '6px',
    fontSize: '13px',
    cursor: 'pointer',
    marginTop: '8px'
  },
  tarjetaPerfil: {
    backgroundColor: '#FFF',
    padding: '14px',
    borderRadius: '8px',
    border: '1px solid #E5E7EB',
    display: 'flex',
    alignItems: 'center',
    gap: '12px'
  },
  avatar: {
    fontSize: '30px',
    backgroundColor: '#E8F5E9',
    borderRadius: '50%',
    padding: '10px'
  },
  botonCuentaRapida: {
    backgroundColor: '#F3F4F6',
    border: '1px solid #D1D5DB',
    padding: '10px',
    borderRadius: '6px',
    textAlign: 'left',
    fontSize: '13px',
    cursor: 'pointer'
  },
  modalOverlay: {
    position: 'fixed',
    top: 0,
    left: 0,
    right: 0,
    bottom: 0,
    backgroundColor: 'rgba(0,0,0,0.5)',
    display: 'flex',
    alignItems: 'center',
    justifyContent: 'center',
    padding: '14px
